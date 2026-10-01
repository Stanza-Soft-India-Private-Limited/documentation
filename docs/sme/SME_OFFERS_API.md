# SME — Promotional Offers API Guide

> Changed 2026-09-21 — exam dimension on users, orders, offers, banners; two BREAKING calls (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).
> Changed 2026-09-21 — per-exam broadcast, validated segment exam (+ fix), survey targetExams, feedback ?exam= (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

Create, schedule and kill promotional pricing campaigns without an app release. When a campaign is
live, users who tap *Try Premium* (or open a share link) see the campaign paywall; when it is not,
they see the normal pricing screen. We expose the endpoints — the SME team builds the UI.

**A campaign now changes what customers are actually charged.** Read §0 before authoring one: a
single field (`priceInPaise`) is real money, and iOS needs an App Store Connect offer code
configured alongside it (§9) or iPhone users never see the discount.

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Authentication:** `x-api-key: <API_KEY_SECRET>` on every `/sme/*` route
**Swagger:** `/api/docs` → tag **SME**

---

## 0. The mental model (read this first)

**A campaign is shown only when BOTH gates pass:**

1. `status === 'LIVE'`, and
2. `startsAt <= now <= endsAt`.

The status is the **kill switch**. `POST /sme/offers/:id/end` pulls a campaign instantly without
touching its dates, and it takes effect on the very next paywall request — there is no cache in
front of the resolver.

**Lifecycle:** `DRAFT → SCHEDULED → LIVE → ENDED → ARCHIVED`, plus `PAUSED` off to the side.

**Reversible vs terminal — the distinction that matters most in the UI:**

| Action | Result | Reversible? |
|---|---|---|
| `POST :id/pause` | `PAUSED` | ✅ `activate` brings it straight back, dates untouched |
| `POST :id/end` | `ENDED` | ❌ **terminal** — can never be re-activated |
| `DELETE :id` | `ARCHIVED` | ❌ **terminal** — the code stays reserved forever |

A new campaign is always created as `DRAFT`. **Pause is what SME should reach for** when a campaign
needs pulling to fix a price or some copy; `end` and `archive` are one-way doors. Please make that
obvious in the portal — an accidental "End now" cannot be undone.

A `PAUSED` campaign **releases its date window**, so a replacement can go live in the same period.
That means resuming re-runs the overlap check and can be refused if something else took the slot.

**Only one campaign may occupy any instant — PER EXAM.** The overlap rule applies to `SCHEDULED`
and `LIVE` campaigns only, so you can freely author several competing **drafts** for the same
period and activate whichever you pick. The check runs transactionally (under a Postgres advisory
lock, keyed on the exam) on activate, and again if you move a live campaign's window — so two
portal users cannot race two overlapping campaigns live.

**A campaign belongs to exactly one exam** (`examId` — **required on create since
2026-09-21, no default**, see §3), so UPSC and APPSC can
each run their own campaign at the same time. It is one exam rather than many because a plan
carries an **absolute** `priceInPaise`, not a percentage: the same campaign applied to an exam
with a lower list price would be a price *increase*, and the discount-floor check would reject it
at checkout — after the user had already tapped buy. Moving a campaign to another exam re-runs
both the overlap check and the discount floor against the target exam's prices.

**The code is public and permanent.** `code` is the token in the share link
(`https://app.prepmonkey.com/open/offer/INDE50`). It is normalised to uppercase, matched
case-insensitively, and **immutable after creation** — links already circulating on WhatsApp must
not break. To change it, archive the campaign and create a new one.

**A stale link is never a dead end.** An unknown, expired, draft or archived code resolves to the
standard paywall rather than an error, because links reliably outlive campaigns.

**Users who already pay are never shown an offer.** The resolver checks premium status first and
returns the standard paywall regardless of the code.

**A campaign carries two different kinds of price, and only one of them is money.** `price`,
`strikePrice`, `period`, `strikePeriod`, `subtitle` and `badge` are **display strings**: rendered
verbatim on the screen and *never* parsed into an amount. **`priceInPaise` is the only field that
changes what a customer is charged** — an integer in paise, so `499000` means ₹4,990.

- **Omit `priceInPaise` and the campaign is presentation-only.** The screen advertises a discount
  and checkout charges the standard price. That is a legitimate mode (a "look what's coming"
  campaign), but it is almost never what you want — it means a user who reads "₹4,990" pays the
  full price.
- **It must be *below* the standard plan price.** A campaign priced at or above standard is refused
  as a discount: the backend logs an error and charges standard. This exists so a typo
  (`5900000` instead of `590000`) cannot charge a user *more* than the normal price while the screen
  tells them they are saving money.
- **Keep the display string and `priceInPaise` in agreement yourself.** Nothing cross-checks them.
  `"price": "₹4,990"` with `"priceInPaise": 199000` will happily show ₹4,990 and charge ₹1,990.

**Discounts are yearly-only.** Author `priceInPaise` on the yearly tier only. The monthly tier is
listed at the standard price with no `strikePrice` and no `priceInPaise`, so the campaign screen
shows it plainly with no strikethrough. There is no monthly discount to configure.

**The discount shape is "pay up front, then renew at standard"** — but it is delivered differently
on each platform, and the difference is visible to customers.

| | Android / web | iOS |
|---|---|---|
| A campaign purchase is | a **one-time payment** at the campaign price | a **subscription** with an App Store Connect **Offer Code** |
| After the first year | does **not** auto-renew; the customer is reminded and re-buys at standard | auto-renews at the standard price |
| A standard-price purchase is | a real auto-renewing subscription | a subscription |

**Why Android campaigns are one-time.** Razorpay cannot charge a discounted first cycle and then
renew at standard. Verified against their live API on 2026-08-06: the first invoice is created
inside the subscription-create call (so an upfront addon always arrives too late to be billed), and
`PATCH /subscriptions/:id` is rejected outright for UPI mandates — which is how most Indian
customers pay. Rather than advertise a renewal behaviour we cannot honour, a campaign is sold as a
single payment. Apple has no such limitation, so iOS keeps a proper discounted subscription.

**Every checkout path reads a campaign now.** The Razorpay **one-time order** flow re-resolves the
campaign server-side and charges `priceInPaise`, recording `offerCode` and the list price on the
order so campaign revenue stays attributable. The Razorpay **subscription** flow is standard-price
only and will **refuse** a request carrying a campaign code, so a campaign can never be silently
charged at full price. iOS applies the discount through the ASC offer code. The client decides the
campaign code it sends, but never the amount — the server re-resolves it, so a user cannot claim a
discount they are not entitled to.

**Apple cannot combine a discount with free months**, which is why campaigns discount the price
instead of adding bonus months. Do not author a campaign whose copy promises "12 + 2 months free" —
see `bonusDays` in §3.

**Without an `appleOfferCode`, iPhone users see the STANDARD paywall for that campaign.** This is
deliberate, not a bug: iOS can only discount through an offer code configured in App Store Connect,
and we will not show a price we have no way of charging. Android and web see the campaign either
way. Section §9 is the per-campaign checklist that keeps the two sides in step.

**Dates:** send ISO-8601 with an explicit offset — `2026-08-16T00:00:00+05:30` for midnight IST.
They are stored as UTC instants and compared as instants, so an IST-evening boundary behaves
correctly on our UTC servers. An offset-less string is interpreted as UTC and will be 5½ hours off.

---

## 1. List campaigns

```
GET /sme/offers?status=LIVE&exam=appsc-group-1
```

| Query | Description |
|---|---|
| `status` | Optional. One of `DRAFT \| SCHEDULED \| LIVE \| PAUSED \| ENDED \| ARCHIVED`. |
| **`exam`** | Optional slug. **Omitted returns every exam's campaigns** — the list screen is cross-exam by design now that several can be LIVE at once, and every row carries its own `examId`. **400 on an unknown slug**, never an empty page that reads as "this exam has no campaigns" while a live one sits under the correct spelling. |

Ordered by `startsAt` descending. The response is `{ data[], total }` — **not** paginated,
and `total` is the length of `data`, not a cross-page count.

**What staging returns today**
<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/offers` → **200**, verbatim:

```json
{ "data": [], "total": 0 }
```

> 🔎 **Staging has no campaigns** (creating one is a live promotional object, so none was
> created for this capture). The envelope above is the live one and it is the part that
> surprises people: **there is no `page` / `limit` / `hasMore`** — unlike every list in
> [`SME_PORTAL_API.md`](./SME_PORTAL_API.md), this endpoint is unpaginated and `total` is
> just `data.length`. The row shape below is verified against the DTO.

**Row shape** (from code):
<!-- shape verified against code @4f6622e — staging holds no campaigns, see note above -->
```json
{
  "data": [
    {
      "id": "6f0a...",
      "code": "INDE50",
      "name": "Independence 2026",
      "status": "LIVE",
      "examId": "upsc-cse",
      "startsAt": "2026-08-01T00:00:00.000Z",
      "endsAt": "2026-08-15T18:30:00.000Z",
      "isActiveNow": true,
      "heroImageUrl": "https://prepmonkey-704630444646-ap-south-1-an.s3.ap-south-1.amazonaws.com/offers/....png",
      "bannerImageUrl": null,
      "requiresCode": false,
      "plans": [ /* … */ ],
      "content": { /* … */ },
      "createdAt": "2026-07-28T09:00:00.000Z",
      "updatedAt": "2026-07-28T09:00:00.000Z"
    }
  ],
  "total": 1
}
```

`isActiveNow` is **derived** (status + window, evaluated now) so the portal never has to
re-implement the resolver's rule. A campaign can be `LIVE` but not `isActiveNow` — that is a
scheduled campaign whose window has not opened yet.

`requiresCode` is now returned on every list/detail row (2026-08-21). It was previously accepted
on create/update but never came back on read — the same bug shape as the FREEDOM15 pricing
incident (a field that exists on write but not on read) — so a code-gated campaign was
indistinguishable from a normal one in the portal.

`examId` is likewise read back on every row (2026-09-21). Without it the list is a set of rows
that look like they violate the one-live-campaign rule for no visible reason — **several
campaigns can now be `LIVE` at the same time, one per exam.** Show the exam as a column, and
never treat "two LIVE campaigns" as a data error without comparing their `examId`.

## 2. Get one campaign

```
GET /sme/offers/:id
```

Same object as a list row, `examId` included. **Errors: 404** when the id is unknown.

**Per-campaign conversion lists** hang off the same id and are documented separately:

| Call | What it answers | Guide |
|---|---|---|
| `GET /sme/offers/:id/abandoned` | Reached for this campaign and never paid | [SME_CAMPAIGN_ABANDONED_API.md](./SME_CAMPAIGN_ABANDONED_API.md) §1–§7 |
| `GET /sme/offers/:id/purchased` | Completed a purchase on this campaign, end to end | [SME_CAMPAIGN_ABANDONED_API.md](./SME_CAMPAIGN_ABANDONED_API.md) §7b |

Both are scoped by the campaign row itself, so every user they return belongs to **the
campaign's own exam** — there is no `?exam=` on either, and adding one would be meaningless.

## 3. Create a campaign

```
POST /sme/offers
```

> ## 🔴 BREAKING (2026-09-21) — `examId` is now REQUIRED
>
> It used to default to `upsc-cse`. Omitting it — or sending `""` / whitespace — is now a
> **400**. Captured from staging on 2026-09-21 (backend `f6329e6`) by POSTing an
> otherwise-valid campaign body with `examId` removed — **no campaign was created**:
>
> ```json
> {
>   "success": false,
>   "message": "examId is required (e.g. upsc-cse). Campaigns are per exam.",
>   "error": "Bad Request",
>   "statusCode": 400,
>   "timestamp": "2026-09-21T14:45:30.551Z",
>   "path": "/api/v1/sme/offers",
>   "method": "POST"
> }
> ```
>
> **Why there is no default.** A tier carries an **absolute** `priceInPaise`, so the
> default silently filed APPSC campaigns against **UPSC's** list prices: priced against
> the wrong list, locked against UPSC's one-live-campaign window, and **invisible to
> every APPSC paywall while the portal reported them live.** A campaign is a per-exam
> object; authoring one without saying which exam cannot be done correctly by guessing.
>
> **Portal action:** make `examId` the **first** field on the create form, a select
> populated from `GET /sme/exams`. Do not prefill it with `upsc-cse` — an unchosen default
> is the failure this change removes. `PATCH` is unaffected (tri-state, §4).

| Body field | Type | Description |
|---|---|---|
| `code` | string, required | 3–32 chars, `[A-Za-z0-9_-]`. Uppercased on save. **Immutable afterwards.** |
| `name` | string, required | Internal label, never shown to users |
| **`examId`** | slug, **required** | Which exam this campaign discounts. Decides **which exam's `exam_plans` prices the discount floor is checked against**, and **which exam the one-live-campaign rule is enforced within**. Validated — an unknown slug is a 400 (`offer_campaign.exam_id` has no FK, so a typo would be accepted by the database and then resolve for nobody). |
| `startsAt` | ISO-8601, required | Window opens |
| `endsAt` | ISO-8601, required | Window closes; must be after `startsAt` |
| `heroImageUrl` | string \| null | Public URL — see §7 |
| `bannerImageUrl` | string \| null | Public URL for the Dashboard "Try Premium" banner — a SEPARATE image from `heroImageUrl`. See §7.1 |
| `requiresCode` | boolean, optional | Default `false`. When `true` the campaign is reachable **only** via its exact code and never auto-applies — use it for TEST campaigns (a 99%-off price that must not reach real users) and for link-distributed promos. Returned on every read since 2026-08-21. |
| `plans` | array, required | Pricing tiers, see below |
| `content` | object, required | Screen copy, see below |

**`plans[]`** — each entry:

| Field | Type | Description |
|---|---|---|
| `id` | string, required | `"yearly"` / `"monthly"`. Checkout matches `yearly`/`annual`/`year` and `monthly`/`month`; any other spelling means the tier is never matched and the user is charged standard price |
| `title` | string, required | Pill label on the card |
| `price` | string, required | **Display** price, e.g. `"₹4,990"`. Rendered verbatim, never parsed |
| `period` | string, required | Suffix, e.g. `"/year"` |
| `strikePrice` | string \| null | Struck-through original. Display only |
| `strikePeriod` | string \| null | Suffix for the struck price |
| `subtitle` | string \| null | e.g. `"First year"` |
| `badge` | string \| null | e.g. `"Best value"` |
| `priceInPaise` | int ≥ 0, optional | **The real charge, in paise** — `499000` = ₹4,990. The only field that changes what a customer pays. Applies to the **first period only**. On iOS renewals go back to standard automatically; on Android/web a campaign purchase is one-time and does not renew at all. Must be **below** the standard price for that plan or the discount is refused and standard is charged. Omit it → presentation-only campaign. **Yearly tier only.** Set it to the same amount as the ASC offer code (§9) |
| `appleOfferCode` | string, optional | The App Store Connect **Offer Code** this campaign redeems on iOS. Uppercase `A–Z0–9`, max 64 chars, e.g. `FREEDOM79`. **Omit it and iPhone users get the standard paywall for this campaign** — they never see the discounted screen |
| `appleProductId` | string, optional | The ASC product the offer code belongs to, e.g. `com.prepmonkey.premium.yearly`. Required alongside `appleOfferCode` — an offer code is only meaningful against its product |
| `bonusDays` | **must be `0`** (or omitted) | **Reserved field, not honoured by any checkout path** — extra entitlement days on top of the plan duration, which nothing delivers today. A value `> 0` is a **400**: `plans[i].bonusDays must be 0 — bonus days are not honoured by any checkout path yet, so a non-zero value would advertise free time the customer never receives.` A value outside `0–3650`, or a non-integer, is a different 400. Campaign purchases are one-time Razorpay orders and Apple offer-code subscriptions; neither adds entitlement days, and Apple cannot combine a discount with free months at all. Refusing is deliberate: the alternative is a campaign that advertises "+60 days" and silently delivers none. The field stays in the schema for a future free-trial-shaped campaign — **keep the copy free of "+2 months" claims** |
| `isDefault` | boolean | The plan selected when the screen opens. **At most one** plan may set it |

**`content`** — every field optional; anything omitted falls back to the standard paywall's copy:
`badgeText`, `titleLine1`, `titleLine2`, `highlightWord`, `highlightStyle`
(`NONE|TRICOLOR|CORAL|LAVENDER`), `urgencyTitle`, `countdownLabel`, `benefitsTitle`, `benefits[]`,
`comparisonTitle`, `comparison[]` (`{feature, free, premium}`), `socialProof`, `ctaLabel`.

> ⚠️ `comparison[]` renders as the Free-vs-Premium table. Those numbers are a **claim about what the
> app enforces** — keep them in sync with the real quota caps, or you will publish limits the
> backend does not honour.

**`content.styles`** (optional, added 2026-08-22) — per-field colour/bold override. Object keyed by
field name, value `{ "color"?: "#RRGGBB", "bold"?: boolean }`:

| Key | Type | Notes |
|---|---|---|
| `styles` | object, optional | Keys must be one of: `badgeText`, `titleLine1`, `titleLine2`, `urgencyTitle`, `countdownLabel`, `benefitsTitle`, `comparisonTitle`, `socialProof`, `ctaLabel`. **Not stylable**: `highlightWord` (it already has `highlightStyle`) and the array fields `benefits[]`/`comparison[]` (no per-row styling) |
| `styles.<field>.color` | string, optional | Must match `^#[0-9A-Fa-f]{6}$` (6-digit hex, `#` + RGB, no short form, no named colours) |
| `styles.<field>.bold` | boolean, optional | |

**`content.socialProofAvatars`** (optional, added 2026-08-22) — array of avatar image URLs shown
next to `socialProof`, at most 5. ⚠️ These are **curated images uploaded via §7's media endpoint,
NOT real user photos** — that is a deliberate privacy decision, not a placeholder. Each entry must
be an `https://` URL, ≤500 characters.

> ⚠️ **Both keys silently degrade rather than error.** An unknown `styles` key, a malformed
> `color`, a non-boolean `bold`, a non-`https` or over-long avatar URL, or more than 5 avatars —
> the offending piece is dropped and the save still succeeds (400 is never returned for this). A
> field with neither a valid `color` nor a valid `bold` is dropped entirely; an empty
> `socialProofAvatars` array is dropped entirely. The clients fall back to their own theme colour /
> no avatar row, so a bad value just looks like today's screen, never a broken one. Same
> merge-by-id-on-PATCH protection as every other `content` key applies — omitting `styles` or
> `socialProofAvatars` on an update leaves the stored value untouched; sending one replaces it
> wholesale.

**Full create body** — this is a **request**, not a response, so there is nothing to
capture back from it; a successful create makes a live promotional object and was
deliberately not run on staging. What *was* exercised on staging 2026-09-21 (backend
`f6329e6`) is this exact body with `examId` deleted, which returned the 400 quoted in the
breaking-change box above — so the required-field set below is current against the
deployed validator.
<!-- shape verified against code @4f6622e — request body; the examId rejection was captured, the create was not run -->
```json
// POST /sme/offers
{
  "code": "INDE50",
  "name": "Independence 2026",
  "examId": "upsc-cse",
  "startsAt": "2026-08-01T00:00:00+05:30",
  "endsAt": "2026-08-16T00:00:00+05:30",
  "heroImageUrl": null,
  "plans": [
    { "id": "yearly", "title": "Yearly", "price": "₹4,990", "period": "/year",
      "strikePrice": "₹5,900", "strikePeriod": "/year", "subtitle": "First year",
      "badge": "Best value",
      "priceInPaise": 499000,
      "appleOfferCode": "FREEDOM79", "appleProductId": "com.prepmonkey.premium.yearly",
      "bonusDays": 0, "isDefault": true },
    { "id": "monthly", "title": "Monthly", "price": "₹569", "period": "/month",
      "strikePrice": null, "strikePeriod": null, "bonusDays": 0 }
  ],
  "content": {
    "badgeText": "Independence Day Offer",
    "titleLine1": "Azadi for limits",
    "titleLine2": "Unlimited Prep.",
    "highlightWord": "Azadi",
    "highlightStyle": "TRICOLOR",
    "urgencyTitle": "Hurry up and Grab your spot 🎉",
    "countdownLabel": "Countdown Begins",
    "benefitsTitle": "Member Benefits",
    "benefits": ["Personalised Dashboard & Progress Tracker"],
    "comparisonTitle": "Features",
    "comparison": [{ "feature": "Mnemonics", "free": "3/Day", "premium": "Unlimited" }],
    "socialProof": "Join 5000+ serious aspirants already preparing smarter with PrepMonkey",
    "ctaLabel": "JOIN PREPMONKEY",
    "styles": {
      "titleLine1": { "color": "#FFB4A2", "bold": true },
      "badgeText": { "color": "#FFFFFF" }
    },
    "socialProofAvatars": [
      "https://cdn.prepmonkey.com/avatars/a.png",
      "https://cdn.prepmonkey.com/avatars/b.png",
      "https://cdn.prepmonkey.com/avatars/c.png"
    ]
  }
}
```

Returns the created campaign (status `DRAFT`). Drafts may freely overlap — the
one-live-campaign-per-exam rule is applied on **activate**, not here.

**Errors: 400** **missing `examId`** · **unknown `examId`** · duplicate code (codes are
**globally** unique, not per exam — a share link cannot say which exam it meant) · inverted
window · a plan missing `id`/`title`/`price`/`period` (the message names the index) · more than
one `isDefault` · `bonusDays` non-zero or out of range · `priceInPaise` not a non-negative
integer, or **not below that exam's standard price** for the tier's plan type ·
`appleOfferCode` not uppercase `A–Z0–9` / over 64 chars.
**Audit:** `OFFER_CREATE` (id, code, name, **examId**).

> ⚠️ **The discount floor is checked against `examId`'s own `exam_plans` rows** — the same rows
> checkout charges from — and a `priceInPaise` **at or above** that exam's standard price is now a
> **400 on write**, naming the tier index, the amount and the standard it lost to. Before 2026-09
> this read the `app_config` singleton, i.e. UPSC's prices, so an APPSC campaign priced correctly
> for APPSC could be rejected and one priced *above* APPSC's list accepted and then silently ignored
> at checkout. This is the single strongest reason `examId` had to become required.
>
> Two gaps remain, deliberately: a tier whose `id` names neither a yearly nor a monthly period
> has no list price to compare against and is left alone (it may be presentation-only), and if the
> plan read **fails**, authoring is not blocked. Neither is a reason to skip eyeballing the number.

## 4. Update a campaign

```
PATCH /sme/offers/:id
```

Accepts `name`, `examId`, `startsAt`, `endsAt`, `heroImageUrl`, `bannerImageUrl`, `requiresCode`,
`plans`, `content`.
Tri-state: **omitted** = untouched · **null** = cleared · **value** = set.

**`examId` is editable and tri-state — and it deliberately does NOT default.** Omitting it
leaves the campaign where it is; sending `upsc-cse` by reflex on every save would drag every
edited APPSC campaign back to UPSC. (This is why the required-on-create rule of §3 is *not*
applied here: on PATCH, an omission is unambiguous.)

Moving a campaign to another exam is allowed — unlike `code` — because the usual reason to send
it is that the campaign was authored under the **wrong** exam, and the alternative (archive and
recreate) burns the share code forever. It is not a free move:

- the **discount floor is re-checked against the target exam's** `exam_plans` prices, so a move
  that would turn the discount into a price rise is refused;
- the **one-live-campaign window is re-checked in the target exam**, so a move that would collide
  with a campaign already occupying that window is refused;
- an unknown slug is a 400.

`bannerImageUrl` follows the same tri-state rule as every other scalar field here: a portal form
that has no banner input simply never sends the key, and the stored value survives untouched. Only
an explicit `null` clears it.

Sending `code` is rejected with an explicit message rather than a generic validation error.
Moving the window of a `SCHEDULED`/`LIVE` campaign re-runs the overlap check.

> ✅ **`plans` and `content` are MERGED, not replaced** (since 2026-08-17). A tier in the payload is
> merged into the stored tier with the same `id`: an **omitted** key keeps its stored value, and only
> an **explicit** value — including an explicit `null` — overwrites. `content` merges the same way,
> one level deep. The array you send still defines the **set and order** of tiers, so omitting a tier
> removes it and adding one appends it; inheritance applies only *within* a matched tier.
>
> This exists because the portal's offer form has no `priceInPaise` / `appleOfferCode` /
> `appleProductId` input. Under the old replace-wholesale behaviour, saving an unrelated edit deleted
> those keys and the campaign kept advertising its discounted price while charging the standard one —
> which happened to the live `FREEDOM15` campaign on 2026-08-15 and went unnoticed for two days
> because nothing errors. **A client must never be able to delete a field it doesn't know exists.**
>
> Validation runs on the **merged** result, so an inherited price is still checked against the
> standard price — a partial payload cannot smuggle in an invalid state.
>
> **Corrected 2026-09-21:** the audit snapshot **does** record pricing. `OFFER_UPDATE` stores a
> before/after of code, name, status, **`examId`**, window, hero **and the money-bearing subset of
> every tier** (`id`, `price`, `priceInPaise`, `bonusDays`, `appleOfferCode`). A price edit — and a
> tier silently losing its discount — is therefore reconstructible from `sme_audit_log`. `examId` is
> in there for the same reason: moving a campaign to another exam changes which list price its
> tiers are discounts against, so it is a pricing change even when no number in `plans` moved.
> (The charge itself — order/payment rows — remains the record of truth for what a customer paid.)

**Errors: 400** attempted code change · unknown `examId` · inverted window · overlap **in the
target exam** · invalid plans · a tier no longer below the **target exam's** standard price.
**404** unknown id.
**Audit:** `OFFER_UPDATE` (before/after of code, name, status, examId, window, hero, plans).

## 5. Activate — publish the campaign

```
POST /sme/offers/:id/activate
```

`DRAFT`/`SCHEDULED` → `LIVE`. This is where the one-live-campaign rule is enforced; a clash returns
400 naming the conflicting campaign and its window. Already-`LIVE` is a no-op.

**Errors: 400** window overlaps another campaign · campaign is `ENDED`/`ARCHIVED`. **404** unknown id.
**Audit:** `OFFER_ACTIVATE` (status before/after).

## 6. Pause, end and archive

```
POST   /sme/offers/:id/pause     → PAUSED   (REVERSIBLE — resume with :id/activate)
POST   /sme/offers/:id/end       → ENDED    (terminal)
DELETE /sme/offers/:id           → ARCHIVED (terminal, soft delete)
```

**Pause** is the safe withdrawal: `LIVE`/`SCHEDULED` → `PAUSED`, dates untouched, and
`POST :id/activate` resumes it. Idempotent — pausing an already-paused campaign is a no-op. It
returns 400 for a `DRAFT` (nothing to withdraw) or a terminal campaign.

**End** and **archive** are one-way. Neither hard-deletes: the code stays reserved and old share
links keep resolving — to the standard paywall.

**Audit:** `OFFER_PAUSE`, `OFFER_END`, `OFFER_ARCHIVE`.

## 7. Hero image upload

```
POST /sme/media/upload-url
```

Two-step: we return a short-lived presigned `PUT`, the portal uploads the bytes **directly to S3**,
then you store `publicUrl` on the campaign. Image data never passes through the API.

| Body field | Type | Description |
|---|---|---|
| `filename` | string, required | Used only to build a readable object key |
| `contentType` | string, required | `image/jpeg` \| `image/png` \| `image/webp` |
| `folder` | string, optional | `offers` (default) \| `banners` \| `exam-logos` \| `pyq-figures` (PYQ stem figures → `questionImageUrl`) |

```json
// POST /sme/media/upload-url
{ "filename": "independence-hero.png", "contentType": "image/png", "folder": "offers" }
```
```json
{
  "uploadUrl": "https://prepmonkey-704630444646-ap-south-1-an.s3.ap-south-1.amazonaws.com/offers/8f3e...-independence-hero.png?X-Amz-Signature=...",
  "publicUrl": "https://prepmonkey-704630444646-ap-south-1-an.s3.ap-south-1.amazonaws.com/offers/8f3e...-independence-hero.png",
  "key": "offers/8f3e...-independence-hero.png",
  "expiresInSeconds": 300,
  "maxBytes": 5242880
}
```

Then: `PUT <uploadUrl>` with header `Content-Type: image/png` and the raw bytes as the body. No auth
header on that PUT — the signature *is* the authorisation, and it expires in 5 minutes.

**Object keys are never reused.** Every upload gets a fresh UUID, so replacing artwork always yields
a new URL — there is no CDN invalidation to worry about, and no stale image in a user's cache.

**Errors: 400** unsupported `contentType`. **503** object storage not configured on the server.

## 7.1 The Dashboard banner (new, 2026-08-21)

Separate from the paywall screen itself: `GET /api/v1/paywall/banner` (a **user** endpoint, JWT)
tells the app's Dashboard whether to render a promotional banner, and with what artwork.

```json
{ "show": true, "imageUrl": "https://.../banners/....png", "code": "INDE50", "campaignId": "6f0a..." }
```

`show`, `imageUrl`, `code` and `campaignId` are **always present**; `imageUrl`/`code`/`campaignId`
are `null` when `show` is `false`. Tapping the banner navigates the client to
`/paywall?code=<CODE>` — the exact path a share link already opens.

**To give a campaign a banner:** upload artwork via `POST /sme/media/upload-url` with
`folder: "banners"` (§7), then set `bannerImageUrl` on the campaign (§3/§4). A campaign with no
`bannerImageUrl` never shows a Dashboard banner, even while it is `LIVE` and resolving normally
through `/paywall`.

**Resolution rule — deliberately simpler than the paywall resolver:**
1. A user with a real paid entitlement never sees a banner (same `hasPaidEntitlement` check as
   the paywall — a trial user, including a lapsed one, still sees it; only a genuinely paying user
   does not).
2. Otherwise: the `LIVE` campaign whose window contains now, with a non-null `bannerImageUrl`,
   most-recently-started first. No code is involved — this endpoint is never called with one.

**🔑 `requiresCode` is deliberately ignored here.** A code-gated campaign (§0, §3) still gets a
Dashboard banner if you set `bannerImageUrl` on it — this is an explicit product decision, not an
oversight. The banner IS the distribution channel for that campaign: tapping it sends the user to
`/paywall?code=<CODE>`, the same code-supplied path a WhatsApp share link already uses, so nothing
about `requiresCode`'s safety property (invisible to the no-code auto-apply crowd) is weakened —
the banner is just another way to *hand someone the code*. It is also safe against old app builds:
a client that predates this endpoint never calls it, so it cannot be affected either way. There is
no iOS-offer-code gate on this endpoint (unlike `/paywall`) — the banner is a static image, not a
price; the actual price check still happens when the tap lands on `/paywall`.

**No caching** — same reasoning as `/paywall` itself: the response depends on the calling user's
premium status, so anything keyed on method+url would leak one user's banner to another.

### 7.1.1 There is now a carousel around it (2026-09-17)

`GET /api/v1/paywall/banner` is **unchanged** — same route, same body, same rule — so every
shipped client keeps working exactly as it does today. Nothing below alters it.

New clients call **`GET /api/v1/banners?platform=android`** instead. That endpoint merges
SME-authored, audience-targeted banners (`exam_banners`, see **[SME_BANNERS_API.md](./SME_BANNERS_API.md)**)
with **this** campaign banner into one ordered list, where the campaign appears as
`"kind": "campaign"` at a fixed `priority` of 0 and destination
`https://go.prepmonkey.com/open/offer/<CODE>`.

You do not author the campaign banner there. It is still just `bannerImageUrl` on the
campaign, resolved by the same rule described above — the carousel calls that rule rather
than restating it, precisely so the two surfaces can never disagree about the same
campaign. An authored banner with a **positive** priority leads the campaign; a
**negative** one trails it.

### ✅ Closed 2026-09-21: `GET /sme/offers?exam=` is now wired

The gap noted here previously — `ListOffersQueryDto` declared only `status`, so
`?exam=appsc-group-1` hit `forbidNonWhitelisted` and returned *"property exam should not
exist"* — is **fixed**. `exam` is declared, passed through and **validated**: a real slug
filters, an unknown one is a **400**, and omitting it returns every exam's campaigns.

Build the portal control. A client-side filter on the returned `examId` still works and is
still correct for a page you already have, but it is no longer the only option.

### App Store Connect groups are per exam

Each exam owns its own ASC **subscription group** (`exams.apple_subscription_group_id`) and
each of its plans owns a product (`exam_plans.apple_product_id`, globally unique). So the
`appleOfferCode` / `appleProductId` on a tier must belong to **`examId`'s** product — an
offer code configured against UPSC's product does nothing for an APPSC campaign, and the
mismatch is invisible until an iPhone user is served the standard paywall. Source the
product id from `GET /sme/exams`, for the exam you just selected, not from memory.

## 8. Previewing before launch

The app resolves campaigns through `GET /api/v1/paywall` (a **user** endpoint, JWT). To QA an
unpublished campaign on a real device, append `?code=<CODE>&preview=1` **and** send the SME
`x-api-key` header on that request. Preview bypasses both the status and the window checks.

Without a valid api key the `preview` flag is ignored — otherwise any user could append it and read
unreleased pricing.

## 9. What SME must do per campaign

A discounted campaign is configured in **two places** — App Store Connect and this API — and they
must agree. Do them in this order:

0. **Decide the exam, first.** Everything below is per exam: the ASC subscription group, the
   product the offer code hangs off, and the list price the discount floor is checked against.
   `examId` is required on create and cannot be inferred (§3).
1. **Create the Offer Code in App Store Connect first.** On **that exam's** yearly subscription
   product (`exam_plans.apple_product_id`, inside `exams.apple_subscription_group_id`), add a
   one-time-use-free **Offer Code** with a **pay-up-front** discount for the first year at the
   campaign price. Cover new, active and lapsed customers — one offer code does all three.
2. **Put that code on the campaign.** Set `appleOfferCode` to the ASC code (uppercase `A–Z0–9`,
   e.g. `FREEDOM79`) and `appleProductId` to the product it belongs to, on the **yearly** tier.
3. **Set `priceInPaise` to the SAME amount you configured in ASC.** `499000` for ₹4,990. This is
   what Android and web are charged; ASC is what iOS is charged. They are two independent
   configurations of one number.
4. **Set the display strings to match** — `price`, `strikePrice` and any price mentioned in
   `content`. Nothing validates the copy against the amount.
5. **Preview it** (§8) on a real device before activating, then `POST :id/activate`.

> 🔴 **If ASC and `priceInPaise` disagree, iPhone users are charged the ASC amount while the screen
> shows the campaign amount.** The backend detects the mismatch on the receipt and logs a drift
> alert — but the customer has already paid by then. There is no way to catch this before the
> charge, which is why step 3 is a copy-paste and not a re-derivation. Android and web are always
> charged `priceInPaise`, so a mismatch also means the two platforms charge different prices for the
> same campaign.

**Why Offer Codes and not Apple's other offer types.** Apple's *introductory* offers reach only
customers who have never subscribed; *promotional* offers reach only existing and lapsed ones. Each
covers about half the audience, so a single campaign would need two configurations kept in lockstep
plus server-side cryptographic signing of every promotional offer. **Offer Codes replace both** —
one configuration, all three customer states, no signing. Do not configure introductory or
promotional offers for a campaign; they are not part of this flow.

---

## 10. Who reached for the campaign and never paid

`GET /sme/offers/:id/abandoned` — the recovery/drop-off surface for a campaign.

**Covered in depth in its own guide: [SME_CAMPAIGN_ABANDONED_API.md](./SME_CAMPAIGN_ABANDONED_API.md).**
Read it before building anything on this endpoint — every row carries a `signal` field that is a
*confidence level*, not a category (`ORDER` is certain, `TAP` is inferred and iOS-only), and a UI
that blends the two is misleading.

Its companion, **`GET /sme/offers/:id/purchased`** — everyone who completed a purchase on the
campaign, end to end — is documented in
**[§7b of the same guide](./SME_CAMPAIGN_ABANDONED_API.md#7b-companion-endpoint--who-actually-bought)**.
Attribution is `orders.offer_code = <campaign code>` and nothing else, which is what keeps
standard-price renewals inside the window out of the list. Both endpoints are scoped by the
campaign row, so everyone they return bought (or nearly bought) **the campaign's own exam**.

---

## Common errors

| Status | Cause | Fix |
|---|---|---|
| 400 | `examId is required (e.g. upsc-cse). Campaigns are per exam.` | **BREAKING, §3** — send `examId` on create. There is no default any more |
| 400 | Unknown `examId`, or unknown `?exam=` on the list | Source the slug from `GET /sme/exams` |
| 400 | `plans[n].priceInPaise (…) must be below the standard … price for exam …` | The floor is **that exam's** `exam_plans` price, not UPSC's |
| 400 | `plans[n].bonusDays must be 0` | No checkout path honours bonus days yet; use a price discount |
| 400 | `code` already exists | Codes are globally unique and permanent; pick another |
| 400 | `code is immutable` | Archive the campaign and create a new one |
| 400 | `endsAt must be after startsAt` | Check the offsets — an offset-less string is read as UTC |
| 400 | `Window overlaps live campaign X on exam Y` | Pause or end X first, or choose a non-overlapping window. The rule is **per exam** — a clash in UPSC does not block an APPSC campaign |
| 400 | `is ENDED, which is terminal` | Ended/archived campaigns never come back — use **pause** next time |
| 400 | `plans[n].period is required` | Every plan needs `id`, `title`, `price`, `period` |
| 400 | `priceInPaise must be an integer` | Paise, not rupees, and no decimals — ₹4,990 is `499000` |
| 400 | `appleOfferCode` rejected | Uppercase `A–Z0–9` only, max 64 chars — no spaces, hyphens or lowercase |
| 401 | Missing/invalid `x-api-key` | Check the header name and secret |
| — | Campaign is live but nobody is discounted | `priceInPaise` missing, at/above standard price, or the tier `id` is not one of `yearly`/`annual`/`year`/`monthly`/`month`. All three are server-log-only — the API returns success |
| — | Discount works on Android, iPhone shows standard price | `appleOfferCode`/`appleProductId` missing on the yearly tier, or the ASC offer code is not live yet |
| 404 | Unknown offer id | It may have been archived — list with `?status=ARCHIVED` |
| 503 | Storage not configured | `S3_BUCKET` / `S3_PUBLIC_BASE_URL` not set on the server |

Every write lands in `sme_audit_log` under `OFFER_CREATE`, `OFFER_UPDATE`, `OFFER_ACTIVATE`,
`OFFER_PAUSE`, `OFFER_END` or `OFFER_ARCHIVE`, with before/after snapshots. Responses are **raw** — there is no
`{success, data}` envelope; branch on the HTTP status code.
