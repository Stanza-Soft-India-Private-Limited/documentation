# SME — Promotional Offers API Guide

Create, schedule and kill promotional pricing campaigns without an app release. When a campaign is
live, users who tap *Try Premium* (or open a share link) see the campaign paywall; when it is not,
they see the normal pricing screen. We expose the endpoints — the SME team builds the UI.

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

**Only one campaign may occupy any instant.** The overlap rule applies to `SCHEDULED` and `LIVE`
campaigns only, so you can freely author several competing **drafts** for the same period and
activate whichever you pick. The check runs transactionally (under a Postgres advisory lock) on
activate, and again if you move a live campaign's window — so two portal users cannot race two
overlapping campaigns live.

**The code is public and permanent.** `code` is the token in the share link
(`https://app.prepmonkey.com/open/offer/INDE50`). It is normalised to uppercase, matched
case-insensitively, and **immutable after creation** — links already circulating on WhatsApp must
not break. To change it, archive the campaign and create a new one.

**A stale link is never a dead end.** An unknown, expired, draft or archived code resolves to the
standard paywall rather than an error, because links reliably outlive campaigns.

**Users who already pay are never shown an offer.** The resolver checks premium status first and
returns the standard paywall regardless of the code.

**Prices are display strings, not amounts.** `"₹4990"` is rendered verbatim by the app. The current
build does not charge from these values — the buy flow still uses the configured base pricing. Do
not treat a campaign as a live discount until the payment integration lands.

**Dates:** send ISO-8601 with an explicit offset — `2026-08-16T00:00:00+05:30` for midnight IST.
They are stored as UTC instants and compared as instants, so an IST-evening boundary behaves
correctly on our UTC servers. An offset-less string is interpreted as UTC and will be 5½ hours off.

---

## 1. List campaigns

```
GET /sme/offers?status=LIVE
```

`status` is optional and must be one of `DRAFT | SCHEDULED | LIVE | ENDED | ARCHIVED`.

```json
{
  "data": [
    {
      "id": "6f0a...",
      "code": "INDE50",
      "name": "Independence 2026",
      "status": "LIVE",
      "startsAt": "2026-08-01T00:00:00.000Z",
      "endsAt": "2026-08-15T18:30:00.000Z",
      "isActiveNow": true,
      "heroImageUrl": "https://prepmonkey-704630444646-ap-south-1-an.s3.ap-south-1.amazonaws.com/offers/....png",
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

## 2. Get one campaign

```
GET /sme/offers/:id
```

Same object as a list row. **Errors: 404** when the id is unknown.

## 3. Create a campaign

```
POST /sme/offers
```

| Body field | Type | Description |
|---|---|---|
| `code` | string, required | 3–32 chars, `[A-Za-z0-9_-]`. Uppercased on save. **Immutable afterwards.** |
| `name` | string, required | Internal label, never shown to users |
| `startsAt` | ISO-8601, required | Window opens |
| `endsAt` | ISO-8601, required | Window closes; must be after `startsAt` |
| `heroImageUrl` | string \| null | Public URL — see §7 |
| `plans` | array, required | Pricing tiers, see below |
| `content` | object, required | Screen copy, see below |

**`plans[]`** — each entry:

| Field | Type | Description |
|---|---|---|
| `id` | string, required | `"yearly"` / `"monthly"` |
| `title` | string, required | Pill label on the card |
| `price` | string, required | Display price, e.g. `"₹4990"` |
| `period` | string, required | Suffix, e.g. `"/year"` |
| `strikePrice` | string \| null | Struck-through original |
| `strikePeriod` | string \| null | Suffix for the struck price |
| `subtitle` | string \| null | e.g. `"12+2 Months"` |
| `badge` | string \| null | e.g. `"Best value"` |
| `bonusDays` | int 0–3650 | Extra entitlement days. Honoured literally on Android/web once payments land; on iOS it is **display-only** — Apple owns the billing period, so a matching App Store Connect offer must be configured separately |
| `isDefault` | boolean | The plan selected when the screen opens. **At most one** plan may set it |

**`content`** — every field optional; anything omitted falls back to the standard paywall's copy:
`badgeText`, `titleLine1`, `titleLine2`, `highlightWord`, `highlightStyle`
(`NONE|TRICOLOR|CORAL|LAVENDER`), `urgencyTitle`, `countdownLabel`, `benefitsTitle`, `benefits[]`,
`comparisonTitle`, `comparison[]` (`{feature, free, premium}`), `socialProof`, `ctaLabel`.

> ⚠️ `comparison[]` renders as the Free-vs-Premium table. Those numbers are a **claim about what the
> app enforces** — keep them in sync with the real quota caps, or you will publish limits the
> backend does not honour.

```json
// POST /sme/offers
{
  "code": "INDE50",
  "name": "Independence 2026",
  "startsAt": "2026-08-01T00:00:00+05:30",
  "endsAt": "2026-08-16T00:00:00+05:30",
  "heroImageUrl": null,
  "plans": [
    { "id": "yearly", "title": "Yearly", "price": "₹4990", "period": "/year",
      "strikePrice": "₹5990", "strikePeriod": "/year", "subtitle": "12+2 Months",
      "badge": "Best value", "bonusDays": 60, "isDefault": true },
    { "id": "monthly", "title": "Monthly", "price": "₹421", "period": "/month",
      "strikePrice": "₹569", "strikePeriod": "/Month", "bonusDays": 0 }
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
    "ctaLabel": "JOIN PREPMONKEY"
  }
}
```

Returns the created campaign (status `DRAFT`).
**Errors: 400** duplicate code · inverted window · a plan missing `id`/`title`/`price`/`period`
(the message names the index) · more than one `isDefault` · `bonusDays` out of range.
**Audit:** `OFFER_CREATE` (id, code, name).

## 4. Update a campaign

```
PATCH /sme/offers/:id
```

Accepts `name`, `startsAt`, `endsAt`, `heroImageUrl`, `plans`, `content`. Tri-state:
**omitted** = untouched · **null** = cleared · **value** = set.

Sending `code` is rejected with an explicit message rather than a generic validation error.
Moving the window of a `SCHEDULED`/`LIVE` campaign re-runs the overlap check.

**Errors: 400** attempted code change · inverted window · overlap · invalid plans. **404** unknown id.
**Audit:** `OFFER_UPDATE` (before/after of code, name, status, window, hero).

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
| `folder` | string, optional | `offers` (default) \| `banners` |

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

## 8. Previewing before launch

The app resolves campaigns through `GET /api/v1/paywall` (a **user** endpoint, JWT). To QA an
unpublished campaign on a real device, append `?code=<CODE>&preview=1` **and** send the SME
`x-api-key` header on that request. Preview bypasses both the status and the window checks.

Without a valid api key the `preview` flag is ignored — otherwise any user could append it and read
unreleased pricing.

---

## Common errors

| Status | Cause | Fix |
|---|---|---|
| 400 | `code` already exists | Codes are globally unique and permanent; pick another |
| 400 | `code is immutable` | Archive the campaign and create a new one |
| 400 | `endsAt must be after startsAt` | Check the offsets — an offset-less string is read as UTC |
| 400 | `Window overlaps live campaign X` | Pause or end X first, or choose a non-overlapping window |
| 400 | `is ENDED, which is terminal` | Ended/archived campaigns never come back — use **pause** next time |
| 400 | `plans[n].period is required` | Every plan needs `id`, `title`, `price`, `period` |
| 401 | Missing/invalid `x-api-key` | Check the header name and secret |
| 404 | Unknown offer id | It may have been archived — list with `?status=ARCHIVED` |
| 503 | Storage not configured | `S3_BUCKET` / `S3_PUBLIC_BASE_URL` not set on the server |

Every write lands in `sme_audit_log` under `OFFER_CREATE`, `OFFER_UPDATE`, `OFFER_ACTIVATE`,
`OFFER_PAUSE`, `OFFER_END` or `OFFER_ARCHIVE`, with before/after snapshots. Responses are **raw** — there is no
`{success, data}` envelope; branch on the HTTP status code.
