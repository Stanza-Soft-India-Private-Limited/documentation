# Campaign form — what to add, change and remove

**Audience:** whoever owns the SME portal's `/offers` page (list, **New campaign**, and **Edit**).
**Observed against:** portal **v2.37.1**, 2026-08-22, editing the live `FREEDOM15` campaign.
**Contract re-verified against backend `8249ea6` (deployed) and mobile `2ea8eaf` on 2026-08-22.**
**Scope:** the form and the list page only. The API contract itself lives in
[`SME_OFFERS_API.md`](./SME_OFFERS_API.md) — this doc does not repeat it, it tells you which parts of
it the form is currently not exposing.

> **Invoke the `frontend-design` skill before building the new controls.** This form drives the
> highest-stakes screen in the product commercially, and three of the additions below are colour and
> image inputs where a careless default is worse than no control at all.

---

## 0. Why this doc exists

The form is missing **four fields the API already accepts**, and one of them — `priceInPaise` — is the
only field that decides what a customer is actually charged.

Because the form has no input for it, a normal "Save changes" used to send a `plans` array **without**
`priceInPaise`, and the backend replaced the array wholesale. The live campaign's real price was
silently deleted **three times** in August 2026. Buyers were quoted **₹5,900 against a ₹4,999 screen**
for two days.

The backend now merges field-by-field instead of replacing, so a save can no longer delete what the
form does not send. **That is a safety net, not a fix.** Until the form can *set* `priceInPaise`, a
campaign's real price still has to be set by a direct API call, and nobody using this portal can tell
whether the discount they just approved is real.

---

## 1. REMOVE — two banners that are now false and actively dangerous

Both of these predate campaign pricing. They are wrong, and they invite an operator to approve a real
discount believing it is cosmetic.

### 1.1 On the `/offers` list page
> *"…Prices are display strings — the buy flow still charges the configured base pricing. Stale or
> unknown codes never error; they fall back to the standard paywall."*

**Delete the pricing sentence.** Keep the rest of that paragraph — the LIVE-window rule and the
stale-code fallback are both still true and both useful.

### 1.2 Inside the **PLANS** section of the create/edit form
> *"Prices are display strings rendered verbatim — the buy flow still charges the configured base
> pricing, so a campaign is not a live discount until the payment integration lands."*

**Delete it entirely.** The payment integration landed on 2026-08-07. Checkout charges
`priceInPaise`. Replace it with the helper text in §3.1 below.

---

## 2. ADD — Identity & window

### 2.1 `requiresCode` — a toggle. **This is the most important addition on the page.**

| | |
|---|---|
| Control | Toggle / checkbox |
| Label | **"Only reachable with the share link"** |
| Default for a NEW campaign | **ON** |
| API field | `requiresCode` (boolean, already accepted; already returned by `GET /sme/offers/:id`) |

Helper text, verbatim:
> *ON — only people who open the share link see this campaign. OFF — **every** non-paying user is
> auto-shown this price, including users on app versions that have never heard of this campaign.*

**Why this matters more than it looks.** On Android and web the app never sends a price — it asks for
a plan and the server prices it. So a LIVE campaign with this OFF discounts the **entire user base**,
silently, including old app builds. That has happened once already. Default it ON and make turning it
OFF a deliberate act.

⚠️ **It gates auto-application, NOT visibility — and the dashboard banner ignores it entirely.**
`GET /paywall/banner` deliberately does **not** check `requiresCode`: the banner *is* the distribution
channel for a code-gated campaign, and tapping it routes to `/paywall?code=<CODE>`, the same path the
share link uses. So a campaign with `requiresCode: ON` **and** a `bannerImageUrl` (§4) is still
advertised on the dashboard to every non-paying user. If the intent is "genuinely only the people I
send the link to" — a test campaign, say — then leave `bannerImageUrl` blank as well. Word the helper
text accordingly; "only people who open the share link see this campaign" is true only with no banner.

---

## 3. CHANGE — the PLANS section

### 3.1 ADD `priceInPaise` — required for any real discount

| | |
|---|---|
| Control | Number input, one per plan, next to `Price` |
| Label | **"Amount actually charged (paise)"** |
| API field | `priceInPaise` (integer, already accepted) |

Helper text, verbatim:
> *`Price` is the text shown on screen. **This** is the money. ₹4,999 = `499900`. Leave blank and the
> customer is charged the standard price no matter what the screen says.*

**Validation you should enforce in the form:**
- Warn loudly if `priceInPaise` is blank while `Price` differs from the standard price — that is
  exactly the silent-mismatch case that caused the incident.
- Warn if `priceInPaise` does not correspond to the rupee figure typed into `Price`. Do not block —
  a deliberate mismatch is occasionally valid — but make it visible.
- `0` is not a discount, it is free. Treat it as a hard confirm.

### 3.2 ADD `appleOfferCode`

| | |
|---|---|
| Control | Text input, one per plan |
| Label | **"App Store offer code"** |
| API field | `appleOfferCode` (string, already accepted) |

Helper text:
> *Created in App Store Connect, not here. Without it, iOS users see the standard price even while
> Android and web get the discount.*

**Format is enforced and rejects hard — validate in the form.** The backend applies Apple's own rule,
`^[A-Z0-9]{1,64}$`: **uppercase letters and digits only**, no spaces, hyphens or punctuation. Unlike
the `styles` fields, a bad value here is **not** silently dropped — it returns a **400 and the whole
save fails**. Upper-case the input as the operator types, and reject anything else before submit.

### 3.2b ADD `appleProductId`

| | |
|---|---|
| Control | Text input, one per plan |
| Label | **"App Store product ID"** |
| API field | `appleProductId` (string, max 200, already accepted) |

The sibling of `appleOfferCode` and equally unexposed today. It is how an incoming Apple transaction
is matched back to this tier for price-drift detection. FREEDOM15 has it set
(`com.stanzasoft.upscbuddy.premium.annual`) only because it was written by direct API call. Same
reasoning as `priceInPaise`: while the form cannot set it, no operator can tell whether it is right.

### 3.3 REMOVE `Bonus days`

The field is accepted by the DTO, carried through every layer, and **rendered nowhere on any
platform**. The server also **rejects any non-zero value outright**. It is a dead control that
currently implies a promise the product cannot keep.

Delete the input. If it is ever implemented, it comes back with a real entitlement behind it.

---

## 4. ADD — a second image, and rename the section

Rename **HERO IMAGE** → **IMAGES**, containing two uploads.

| | Hero | **Banner (NEW)** |
|---|---|---|
| API field | `heroImageUrl` (existing) | `bannerImageUrl` |
| Where it appears | Inside the campaign paywall | On the app **dashboard**, below the streak card |
| Aspect ratio | as today | **5:2** |
| Recommended size | as today | **1500 × 600** |
| Blank means | flat colour fallback | **no banner is shown at all** |

Both use the **existing** `POST /api/v1/sme/media/upload-url` endpoint — no new API. Pass
`folder: "banners"` for the banner (that folder value is already accepted). Same limits as hero:
JPEG / PNG / WebP, max 5 MB, presigned S3 PUT, fresh key every upload so there is never a stale cache.

Helper text for the banner:
> *Shown on the dashboard to every non-paying user while the campaign is live, and taps straight
> through to this campaign's paywall. Leave blank for no banner. Crops to 5:2 — put nothing important
> in the top or bottom edge.*

✅ **CORS is configured — verified end to end on 2026-08-22.** An earlier version of this doc listed
a missing S3 CORS rule as a blocker. It is not. Proven against production, browser-style: presign →
`OPTIONS` preflight (`Origin: https://sme.prepmonkey.ai`, `Access-Control-Request-Method: PUT`) → 200
with `Access-Control-Allow-Origin: *` and `Access-Control-Allow-Methods: PUT` → real `PUT` → 200 →
anonymous `GET` of `publicUrl` → 200. The rule is origin-`*`, so the portal, the web app and
localhost are all covered. Nothing to request from infra.

---

## 5. ADD — colour and bold per copy field

Today the whole **SCREEN COPY** section is text-only, and every colour and weight is hard-coded in the
apps. The one exception is `Highlight style` (`NONE` / `TRICOLOR` / `CORAL` / `LAVENDER`), which
applies to a single word inside Title line 1.

Add, **next to each of these nine fields**, a colour swatch and a bold toggle:

`Badge text` · `CTA label` · `Title line 1` · `Title line 2` · `Urgency title` · `Countdown label` ·
`Benefits title` · `Comparison title` · `Social proof`

Deliberately **not** stylable, do not add controls for them: `Highlight word` (it already has
`Highlight style`), and the `Benefits` / `Free vs Premium` list rows (per-row styling is out of scope).

**API shape** — a new optional `styles` object inside `content`:

```jsonc
"content": {
  "titleLine1": "Azadi from limits",
  "styles": {
    "titleLine1": { "color": "#FFB4A2", "bold": true },
    "badgeText":  { "color": "#FFFFFF" }
  }
}
```

Rules the backend enforces, so the form should mirror them:
- `color` must be `#RRGGBB` (6-digit hex). Anything else is **silently dropped** and the app falls
  back to its built-in colour — it will not error, so a typo fails quietly. Validate in the form.
- `bold` is a boolean. Omit rather than sending `false` if you mean "leave as-is".
- Unknown field names are dropped silently.
- Omitting `styles` entirely on a PATCH does **not** delete existing styles.

🔴 **The one thing that will bite you — `styles` is replaced WHOLESALE, not merged per field.**
`plans` merges tier-by-tier and key-by-key (that was the fix for the price-wipe incident in §0), but
`content` merges only at the **top level**. `styles` is a single top-level key, so a PATCH sending

```jsonc
"content": { "styles": { "titleLine1": { "color": "#FFFFFF" } } }
```

**deletes every other style entry** — `badgeText`, `ctaLabel`, all of them. This is the exact same
failure mode as the `priceInPaise` incident, on a field that has no safety net.

**Therefore: the form must send the COMPLETE `styles` object on every save**, rebuilt from all nine
controls, not just the field the operator touched. Read the current value from
`GET /sme/offers/:id`, apply the edit, send the whole object back.

This also changes the "reset to default" rule below: clearing a field's colour means **omitting that
field from the styles object entirely** while still sending the other eight — not sending `#000000`,
and not sending `styles` with only that field in it.

**Two things to build in, not optional:**
1. **A "reset to default" affordance per field.** Once an operator sets a colour there is currently no
   way back to the app's default from the API shape alone — clearing the swatch must send the field
   with no `color` key, not `#000000`.
2. **A contrast warning.** The paywall hero is near-black. A dark colour here produces invisible text
   and there is no preview accurate enough to catch it (see §6). Warn below roughly 4.5:1 against the
   hero background.

---

## 6. FIX — the paywall preview is lying in two ways

The live preview on the right of the form is genuinely useful, and it is currently wrong about the
countdown.

1. **It shows a seconds cell.** The app has **no seconds field at all** — it renders days, hours and
   minutes only, on both mobile and web. Remove `SEC` from the preview.
2. **It shows no separator between cells.** The app puts a character between them. As of the app
   release currently in flight that character is **`:`** (it was `•`). Match it.

While you are in there: the preview renders the plan cards side by side, and the **monthly card looks
hollow** because `Subtitle`, `Badge`, `Strike price` and `Strike period` are blank for that tier while
yearly fills all four. The app-side layout is being reworked for this, but the preview will keep
looking sparse until the fields are filled. Consider showing a hint on any plan whose optional fields
are all empty: *"This tier will render with empty space where the badge/subtitle/strike price would be."*

---

## 7. Field-by-field summary

| Section | Field | Action | API field |
|---|---|---|---|
| List page | "Prices are display strings…" banner | **REMOVE** | — |
| Identity & window | Only reachable with share link | **ADD** (toggle, default ON) | `requiresCode` |
| Images | Section rename HERO IMAGE → IMAGES | **CHANGE** | — |
| Images | Banner image upload, 5:2 / 1500×600 | **ADD** | `bannerImageUrl` |
| Plans | "Prices are display strings…" banner | **REMOVE** | — |
| Plans | Amount actually charged (paise) | **ADD** | `priceInPaise` |
| Plans | App Store offer code (uppercase A-Z/0-9 only; bad value = 400) | **ADD** | `appleOfferCode` |
| Plans | App Store product ID | **ADD** | `appleProductId` |
| Plans | Bonus days | **REMOVE** | `bonusDays` |
| Screen copy | Colour + bold on 9 text fields (send the WHOLE object every save) | **ADD** | `content.styles` |
| Screen copy | Social-proof avatar images (up to 5, 3 shown) | **ADD** | `content.socialProofAvatars` |
| Preview | Seconds cell in countdown | **REMOVE** | — |
| Preview | `:` separator between countdown cells | **ADD** | — |

### On `socialProofAvatars`
Up to **5** https URLs uploaded through the same media endpoint; the app renders **three** of them as
the small overlapping circles beside the social-proof line (today those are flat lavender dots).
URLs must start with `https://` and be ≤500 characters; anything else is dropped silently, and the
list is capped at 5 by filter-then-slice. ℹ️ If more than three are supplied the app **shuffles and
picks three at random** — so do not assume the first three, or a stable order, will be the ones shown.

🔑 **These are curated brand images. They are explicitly NOT real users' profile photos** — that was a
deliberate decision, because putting real aspirants' faces on a sales screen is a purpose they never
consented to. Do not add a "pull from real users" option.

---

## 8. Definition of done

- [ ] Both false pricing banners are gone from the list page and the PLANS section.
- [ ] `requiresCode` is a visible toggle, defaulting ON for a new campaign.
- [ ] Every plan row has `priceInPaise` and `appleOfferCode`, and `Bonus days` is gone.
- [ ] A save that changes only the copy still round-trips `priceInPaise` unchanged — verify by saving
      `FREEDOM15` and re-reading `GET /sme/offers/:id`, confirming the yearly tier still shows
      `priceInPaise: 499900` and `appleOfferCode: "FREEDOM15"`.
- [ ] Banner upload works end to end (CORS is already in place — see §4).
- [ ] Colour swatches reject anything that is not `#RRGGBB`, warn on low contrast, and can be cleared
      back to default.
- [ ] A save that changes ONE colour leaves the other eight intact — verify by setting `titleLine1`
      and `ctaLabel` in separate saves, then re-reading `GET /sme/offers/:id` and confirming BOTH are
      still present. This is the `styles`-is-replaced-wholesale trap in §5.
- [ ] The preview shows no seconds cell and uses `:` between countdown cells.
