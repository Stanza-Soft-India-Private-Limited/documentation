# SME Banners API — the dashboard carousel

> Changed 2026-09-21 — exam-scoping gaps closed (existing contracts).
> Changed 2026-09-21 — exam dimension on users, orders, offers, banners; two BREAKING calls (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

Managing the promotional images that appear on an exam's home screen.

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Auth:** `x-api-key: <API_KEY_SECRET>` on every `/sme/*` request.

> **There is no response envelope.** Success bodies are raw; errors are shaped by the
> exception filter. Branch on the HTTP status code, never on a `success` field.

> **Invoke the `frontend-design` skill before building any screen from this doc.**

---

## 1. The whole mental model, in one page

**A banner is one image in the dashboard carousel of one exam.** It carries no copy of
its own: everything the user reads is baked into the artwork, and the only interaction is
a tap that opens somewhere inside the app.

Four things decide whether a given user sees it, and **all four must hold**:

| | |
|---|---|
| **status** | must be `LIVE`. This is the kill switch — pulling a banner is one call, no date editing, no deploy. `DRAFT` → `LIVE` (activate) → `PAUSED` (reversible) → `ENDED` (**terminal**). |
| **window** | `startsAt <= now <= endsAt`, inclusive at both ends. |
| **audience** | `tier × platform × account-age`, all three. |
| **exam** | always required. A banner belongs to exactly one exam. |

### Audience, exactly as implemented

**tier** — the user's commercial standing *in this exam*, computed server-side from the
entitlement row and never asserted by the client. The order of the test is the contract:

1. `paid` — a **live** entitlement row for this exam. Checked first: a paying customer is
   a paying customer whatever their trial dates say.
2. `trial` — no live entitlement, but inside the trial window. The clock is person-level
   (one lifetime window from signup); the **length** is this exam's `trialDays`
   (`SME_EXAMS_API.md` §9). A churned account is not on trial however recently it signed
   up, and an SME per-user trial extension counts here.
3. `lapsed` — an entitlement row exists but has expired or been revoked. **Deliberately
   below `trial`:** someone who lapsed while still inside their signup window is, for the
   purpose of what to advertise, still a trial user. *Trial outranks lapsed.*
4. `free` — no row at all and no trial. The default, and the largest audience.

These are **per exam**: a UPSC subscriber is `free` in APPSC. An exam with `trialDays = 0`
(which `appsc-group-1` is, since the 2026-09-17 migration) has **no `trial` users at all**,
so targeting only `trial` there reaches nobody.

**platform** — `android` | `ios` | `web`, stated by the calling client. It is a
**required** query param on the app endpoint: unlike `?exam=`, there is no safe default,
and guessing would silently serve another platform's audience.

**account age** — **whole days** since signup, floored: a user is age `0` for their entire
first day and becomes `1` exactly 24 h in. Both bounds are **inclusive**, and `null` means
unbounded on that side. So `min: 0, max: 7` reaches a user for their first **eight** days,
and `min: 7` reaches them on their seventh day. A missing or future-dated `createdAt`
degrades to age 0 rather than dropping the user out of every age-bounded audience.

Omit an audience field on create and it means *everyone* — all four tiers, all three
platforms, any age. An **empty** array is not authorable (the DTO requires a non-empty
one) and, were one written by hand in SQL, would match **nobody**, not everybody.

### Artwork

5:2, author at **1500×600**. The slot is a fixed aspect and width-driven, so anything else
is cropped or letterboxed. Upload via `POST /sme/media/upload-url { "folder": "banners" }`
(`image/jpeg` · `image/png` · `image/webp`) and store the returned public URL. **The
dimensions cannot be checked from a URL — nothing validates them, so get them right.**
`imageUrl` must be `https://…`: an `http://` URL is blocked by ATS on iOS and renders
blank.

### Destination

An **in-app smart link only** — the URL must match
`https://app.prepmonkey.com/open/…` or `https://go.prepmonkey.com/open/…`. Anything else
is a 400 that says so. Those are the only two hosts the app claims (AASA / assetlinks) and
the only paths `DeepLinkTarget.parse` understands; any other URL opens a browser, and a
promotion that leaves the app cannot convert. (`go.` exists as a second host because iOS
will not hand a Universal Link to the app while it is already on that same domain.)

Anything the deep-link router already parses works with no new routing — a paywall, an
offer code, a reel, a document.

### Ordering, and the campaign banner

`priority` **DESC**, then window start **DESC**. Several banners being `LIVE` at once is
normal and intended — the slot is a carousel, so unlike offer campaigns there is no
one-at-a-time rule.

**The live offer campaign's `bannerImageUrl` is injected into the same carousel
automatically, at a fixed `priority` of 0**, for any user who has not yet **paid** for this
exam (a trial user, even a lapsed one, still sees it — that is who it exists to convert).
Its destination is `https://go.prepmonkey.com/open/offer/<CODE>`. You do not create it
here; it belongs to the campaign (`SME_OFFERS_API.md`). Give an authored banner a
**positive** priority to lead it, a **negative** one to trail it. Among candidates tying on
both keys, authored banners lead — they were written for this slot, the campaign banner is
a guest from another surface.

The campaign's eligibility is evaluated by the *campaign's* rule, not this table's: no
audience targeting, `requiresCode` ignored, hidden only from a paying user. `GET /banners`
and `GET /paywall/banner` therefore can never disagree about the same campaign.

### Publishing is a separate call

`status` is **rejected on create and on update** (400, with a message saying what to use
instead). A banner always starts `DRAFT`, and going live is its own audited action — so
"when did this start showing?" is answerable from the trail rather than buried inside a
copy edit. Audited actions: `BANNER_CREATE`, `BANNER_UPDATE`, `BANNER_ACTIVATE`,
`BANNER_PAUSE`, `BANNER_END`.

---

## 2. Status machine

```
        create                activate              pause
  ──────────────▶ DRAFT ──────────────▶ LIVE ◀──────────────▶ PAUSED
                    │                    │    activate            │
                    │                    │                        │
                    └────────── end ─────┴──────── end ───────────┘
                                         ▼
                                       ENDED   (terminal — DELETE lands here too)
```

| Call | From | To | Refused when |
|---|---|---|---|
| `activate` | `DRAFT`, `PAUSED` | `LIVE` | the banner is `ENDED` (terminal), or `endsAt` is already in the past. Idempotent on an already-`LIVE` banner. A window that has not opened yet is fine — it goes live on schedule. |
| `pause` | `LIVE` | `PAUSED` | anything that is not `LIVE`. Idempotent on `PAUSED`. Dates untouched, so `activate` brings it straight back. |
| `end` | any | `ENDED` | never. Idempotent. **Cannot be undone.** |

Activating a banner whose window has already closed is refused on purpose: it would read
`LIVE` in the portal and show to nobody, which turns "I published it and nothing happened"
into a support ticket instead of a 400 naming the date.

---

## 3. Endpoints

### `GET /sme/banners?exam=&status=`

Both filters optional; omitting `exam` returns every exam's banners, each row carrying its
own `examId`.

> ✅ **`exam` is now STRICT** (landed 2026-09-21). An unknown slug is a **400**, not an empty
> page — matching every other SME exam filter. The lenient reading it replaced returned `[]`,
> which on a management screen is indistinguishable from "this exam has no banners" and sent
> an SME off to re-upload artwork that already existed under the correct spelling.
>
> Handle the 400 in the list screen, and source slugs from `GET /sme/exams` rather than
> free text. `status` is unvalidated by contrast — an unknown status simply matches nothing.

**The list is unpaginated.** There is no `page`/`limit`; it returns every matching row as
`{ data, total }`, ordered by `priority` descending then `startsAt` descending, and `total`
is simply `data.length` — not a count of a larger set behind it.

```json
{
  "data": [
    {
      "id": "0b1f…",
      "examId": "appsc-group-1",
      "name": "APPSC launch — week 1",
      "status": "LIVE",
      "imageUrl": "https://…/banners/appsc-launch.png",
      "destinationUrl": "https://go.prepmonkey.com/open/offer/APPSC25",
      "audienceTiers": ["free", "lapsed"],
      "platforms": ["android", "ios"],
      "minAccountAgeDays": 0,
      "maxAccountAgeDays": 7,
      "priority": 10,
      "startsAt": "2026-09-30T18:30:00.000Z",
      "endsAt": "2026-10-14T18:30:00.000Z",
      "isLiveNow": true,
      "createdAt": "…",
      "updatedAt": "…"
    }
  ],
  "total": 1
}
```

`isLiveNow` is **derived** (status `LIVE` **and** the window contains now) so the portal
never re-implements the resolver's rule. A banner can be `LIVE` and not `isLiveNow` — that
is one scheduled for later.

### `GET /sme/banners/:id`
One banner, the same object. `404` if unknown.

### `POST /sme/banners` — creates a **DRAFT**

```json
{
  "examId": "appsc-group-1",
  "name": "APPSC launch — week 1",
  "imageUrl": "https://cdn.prepmonkey.com/banners/appsc-launch.png",
  "destinationUrl": "https://go.prepmonkey.com/open/offer/APPSC25",
  "audienceTiers": ["free", "lapsed"],
  "platforms": ["android", "ios"],
  "minAccountAgeDays": 0,
  "maxAccountAgeDays": 7,
  "priority": 10,
  "startsAt": "2026-10-01T00:00:00+05:30",
  "endsAt": "2026-10-15T00:00:00+05:30"
}
```

`examId`, `name`, `imageUrl`, `destinationUrl`, `startsAt`, `endsAt` are **required**;
everything else defaults (all four tiers, all three platforms, unbounded age, priority 0).

**Dates:** send ISO-8601 with an explicit offset — `2026-10-01T00:00:00+05:30` for midnight
IST. They are stored and compared as UTC instants; an offset-less string is read as UTC and
will be 5½ hours off.

`name` is internal (≤ 120 chars) and is never shown to a user. `priority` is bounded to
−1000…1000 so a typo cannot bury everything.

### `PATCH /sme/banners/:id` — **MERGE**

An omitted field is left untouched. An explicit `null` clears a nullable one
(`minAccountAgeDays`, `maxAccountAgeDays`). Nothing is replaced wholesale, so a portal form
that does not render a field can never delete it — that exact bug shipped on a live
campaign once (`priceInPaise` dropped, ₹4,999 advertised while ₹5,900 was charged, for two
days).

```json
{ "priority": 20, "maxAccountAgeDays": null }
```

The window and the age range are validated against the **merged** pair, so sending only
`min` is still checked against the stored `max`. `examId` *is* re-targetable (unlike an
offer's `code`): the usual reason to send it is that the banner was authored under the
wrong exam, and nothing circulating in the wild refers to it.

### `POST /sme/banners/:id/activate` → `LIVE`
### `POST /sme/banners/:id/pause` → `PAUSED`
### `POST /sme/banners/:id/end` → `ENDED`

No body. Each returns the updated banner. See the table in §2 for what each refuses.

### `DELETE /sme/banners/:id`

**Soft delete: identical to `end`.** A banner is never hard-deleted — audit rows refer to
it, and removing it would make "what was showing on the 3rd?" unanswerable.

### Common errors

| Status | Cause |
|---|---|
| 400 | `status` sent on create or update (publishing is its own call) |
| 400 | unknown `examId` — create the exam first (`POST /sme/exams`) |
| 400 | unknown `?exam=` on the **list** (strict since 2026-09-21; it used to return `[]`) |
| 400 | `destinationUrl` outside the `app.`/`go.prepmonkey.com/open/` allow-list |
| 400 | `imageUrl` not `https://` |
| 400 | empty or out-of-set `audienceTiers` / `platforms` |
| 400 | `endsAt <= startsAt`, or `minAccountAgeDays > maxAccountAgeDays` |
| 400 | `activate` on an `ENDED` banner, or one whose window has closed |
| 400 | `pause` on a banner that is not `LIVE` |
| 400 | any key not in the DTO — the global pipe runs `forbidNonWhitelisted` |
| 404 | unknown banner id |

---

## 4. What the app receives

```
GET /api/v1/banners?platform=android      (JWT + X-Exam)
```

Gated on the `dashboard.banners` feature flag (`SME_EXAMS_API.md` §7): an exam that has
switched its banner slot off gets **403 `FEATURE_DISABLED`**, not an empty list — "off for
this exam" and "nothing matched right now" are different facts and only one is worth
investigating.

```json
{
  "banners": [
    { "id": "…", "kind": "banner",   "imageUrl": "https://…", "destinationUrl": "https://app.prepmonkey.com/open/paywall" },
    { "id": "…", "kind": "campaign", "imageUrl": "https://…", "destinationUrl": "https://go.prepmonkey.com/open/offer/INDE50" }
  ]
}
```

`banners` is **always present** and is `[]` when nothing matches. Every key on every row is
always present. `platform` is required; `?exam=`/`X-Exam` resolve exactly as everywhere
else, falling back to `upsc-cse`.

**Not cached**, deliberately: every dimension of the answer is per user (tier, account
age), and the cache interceptor keys on method + url + query + exam with no user dimension
— one entry would show a paying customer's banners to a free user. Same reasoning as
`/paywall`.

`GET /paywall/banner` is **unchanged** and still serves the campaign banner on its own to
already-shipped clients. New clients should call `GET /banners` and ignore it.
