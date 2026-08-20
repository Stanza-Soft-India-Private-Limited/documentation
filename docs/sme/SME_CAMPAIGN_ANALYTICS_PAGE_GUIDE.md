# SME Campaign Analytics — Page Redesign Brief

> ## ⛔ BEFORE YOU WRITE ANY UI CODE — INVOKE THE `frontend-design` SKILL
> If you are a Claude/coding session rebuilding this page, your **first action** is to invoke the
> **`frontend-design`** skill (`/frontend-design`). Do this **even if no one asked you to**. Every
> card, table, empty state and label below must be designed through that skill — do not reach for a
> default admin-template layout.
>
> Design for what this screen actually is: **the answer to "is this campaign working, and who do we
> chase?"** The reader is a marketer or founder who has just spent money promoting a code and wants
> to know, in about ten seconds, whether anyone bought. Then they want a list they can act on. The
> screen is *a verdict first, a work queue second* — not a metrics wall.

This is the change brief for `/offers/:id/analytics` (portal **v2.37.1**). The page is not broken —
every number on it is faithful to the API. It is **hard to read**, and this document says exactly why
and what to change. API contract lives in
**[SME_CAMPAIGN_ABANDONED_API.md](./SME_CAMPAIGN_ABANDONED_API.md)**; read it alongside this.

---

## Why the current page confuses people

Observed on the live FREEDOM15 screen, with the real numbers:

1. **Four true denominators, no bridge between them.** The page says "17 campaign orders", "9 users
   reached the payment step", "7 certain abandoners" and "12 inferred" within one viewport. All four
   are correct and all four count different things (orders / users / users-after-exclusions /
   users-on-a-different-signal). Nothing on screen explains the 9 → 7 gap, so it reads as a bug.
2. **Two tables that cannot be added together.** Table 1 is Android+web, table 2 is iOS-only and
   inferred. Placed one under the other with similar styling, they look like halves of one list.
3. **The window is not the campaign's real life.** FREEDOM15 went LIVE **10 Aug** but was not
   promoted until **14 Aug**. The page reported from the 10th, so four days of internal setup traffic
   appeared as demand. *(New `from`/`to` params fix this — see §3.)*
4. **The list is mostly staff.** Of the 7 "certain abandoners", effectively all were team accounts
   (`@stanzasoft.com`, plus team members on personal Gmail). A "recovery list · 7 to chase" that
   contains no customers is worse than an empty one.
5. **There was no "just visited" number at all** — the stage most people assume they are looking at.
   *(New `VIEW` signal — see §2.)*

---

## What changed in the API (2026-08-20)

### 1. `VIEW` — the missing first stage
A **fourth signal**, `VIEW`: the user opened the campaign paywall and never created an order. Server
observed on **every platform** (the resolver stamps the campaign it matched), so it is certain, not
inferred. Users are counted at their **furthest** stage, so the signals partition cleanly.

⚠️ **Deploy-forward: no history before 2026-08-20.** It cannot be backfilled. Any window opening
earlier under-reports views *and only views* — you must label this, or the funnel will look like a
catastrophic drop-off that is really just missing capture.

### 2. `funnel` — campaign-wide counts, not page-local
The response now carries a `funnel` object counted over **all** matching users:
`viewed · joined · paid · viewedNotJoined · paidByPlatform · revenuePaise · discountGivenPaise ·
discountUnclaimedPaise · orderCount`. Use these for every headline. `pageCounts` remains page-local
and must never be used for a headline.

### 3. `from` / `to` — scope to the real promotion date
Both clamp **into** the campaign window and can never widen it. Add a date control defaulting to the
campaign start, and let the operator set the day the link actually went out.

### 4. `GET /sme/offers/:id/purchased` — the end of the funnel
Everyone who completed a purchase **on this campaign**, iOS and Android alike. Attribution is
`orders.offer_code = <campaign code>` and nothing else, which is what keeps ordinary Apple renewals
and standard subscriptions out of a campaign number. Carries `source` (APPLE / RAZORPAY),
`amountPaise`, `listAmountPaise`, `savedPaise`, `premiumGrantedAt`, `premiumExpiresAt`,
`capturedPaise`.

---

## Proposed information hierarchy

Four bands, top to bottom. **Verdict → funnel → who to chase → who bought.**

### Band 1 — the verdict (one sentence, not five cards)
Lead with a plain-English line the reader can repeat in a meeting:

> **"₹0 from 0 buyers. 34 people opened the offer, 9 reached checkout, none paid."**

Then at most **three** supporting stats: `revenue` · `buyers` · `discount given`. Today's five
equal-weight cards force the reader to work out which one matters. Do not put "certain abandoners"
and "inferred interest" in the same row as revenue — they are diagnosis, not outcome.

### Band 2 — the funnel (this replaces the confusing prose block)
One horizontal funnel, four stages, each showing **distinct users** and the drop to the next:

```
  Viewed          Joined           Paid
  34  ──────────▶  9  ──────────▶  0
      −25 left        −9 abandoned
   (from 20 Aug)
```

Rules:
- Label the unit **users** on every stage. Show `orderCount` only as a subtitle
  ("9 users · 17 orders"), never as a stage.
- The `viewed` stage carries a persistent "since 20 Aug" badge while `viewedIsDeployForward` is
  true. Do not silently draw a stage the data cannot support for the whole window.
- Each drop is clickable and filters Band 3.

### Band 3 — who to chase (one table, `signal` as a column)
**Merge today's two tables into one**, with a signal chip per row and a filter above it. This is the
single most valuable change: it removes the "can I add these together?" question permanently.

| Chip | Meaning shown on hover | Actionable? |
|---|---|---|
| `VIEW` | Opened the offer, never started checkout | ✅ |
| `ORDER` | Reached the payment sheet, never paid | ✅ highest intent |
| `BOTH` | Both signals | ✅ |
| `TAP` | iOS upgrade tap in-window — **not confirmed campaign intent** | ⚠️ volume only |

Keep the existing iOS warning, but attach it to the `TAP` chip and filter rather than to a whole
section. Default the filter to `VIEW + ORDER + BOTH` (the actionable set) with `TAP` off — the user
opts into the fuzzy data instead of being handed it.

### Band 4 — who bought (new, from `/purchased`)
Usually empty; design the empty state first. When populated: user, platform chip (Apple/Razorpay),
charged, saved, granted-at, expires-at. Show `savedPaise` per row — "what this campaign actually
cost us" is the number nobody can currently answer.

⚠️ `capturedPaise: null` on a PAID row means no captured payment row exists behind it. Render a
warning chip, never a silent fallback to the order amount.

---

## Cross-cutting rules

1. **Add an "exclude internal accounts" toggle, default ON.** Filter `@stanzasoft.com` and
   `@designasylum.in` on the **domain part only** — the same rule already used for marketing exports.
   Show the count that was hidden ("4 internal accounts hidden") so nothing disappears silently.
   Without this the recovery list is unusable.
2. **State the window you are describing**, from `window.from`/`window.to` — not the campaign dates.
   They now differ whenever `from` is set.
3. **Bridge every exclusion.** Where the funnel's `joined` (9) exceeds the actionable rows (7), say
   "2 excluded — already converted". An unexplained gap reads as a bug.
4. **Never sum `ORDER + TAP`** into one headline. If you need a total, label it approximate.
5. **Paise, always.** `499900` = ₹4,999. Never render ₹0 where the value is `null` — use a dash.
6. **Raw responses.** No `{success, data}` envelope on any `/sme/*` route.

---

## Fix on the offers list page too

`/offers` currently shows this banner:

> *"Prices are display strings — the buy flow still charges the configured base pricing."*

**This is false and actively harmful.** Campaign pricing shipped; checkout really does charge the
campaign price (₹4,999 for FREEDOM15, verified against Razorpay's own checkout). Anyone who reads
that banner will distrust a correct price, or worse, assume a discount is cosmetic and approve one
that is real. Delete it.

---

## What will be empty on day one — plan for it

- **`viewed` is 0 for any window before 2026-08-20.** Do not render a funnel that implies nobody
  looked; render the "since 20 Aug" badge, or hide the stage entirely for older windows.
- **`/purchased` is empty for FREEDOM15.** That is the true state, not a failure. The empty state
  should say so plainly and point at the funnel for where people dropped.
- **`platform` may be `null`** for web traffic before 2026-08-16. Render "Unknown", never "Web".

---

## Definition of done

- [ ] One merged table with a `signal` filter; `TAP` off by default
- [ ] Funnel band driven by `funnel.*`, units labelled **users**, deploy-forward badge on `viewed`
- [ ] Verdict sentence above the stats
- [ ] Date scoping wired to `from`/`to`, defaulting to campaign start
- [ ] Internal-domain toggle, default ON, with a hidden-count label
- [ ] `/purchased` band with the `capturedPaise` warning chip
- [ ] Every exclusion gap explained in words on screen
- [ ] The false pricing banner removed from `/offers`
