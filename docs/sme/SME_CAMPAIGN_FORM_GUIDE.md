# Campaign form — what to add, change and remove

**Audience:** whoever owns the SME portal's `/offers` page (list, **New campaign**, and **Edit**).
**Observed against:** portal **v2.37.1**, 2026-08-22, editing the live `FREEDOM15` campaign.
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

⚠️ **Known blocker:** the S3 bucket has **no CORS rule for the portal's origin**, so a browser-side
`PUT` to the presigned URL will fail. This has been outstanding since 2026-07-28. Flag it when you
start — it blocks *both* uploads, not just the new one.

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
| Plans | App Store offer code | **ADD** | `appleOfferCode` |
| Plans | Bonus days | **REMOVE** | `bonusDays` |
| Screen copy | Colour + bold on 9 text fields | **ADD** | `content.styles` |
| Screen copy | Social-proof avatar images (up to 5, 3 shown) | **ADD** | `content.socialProofAvatars` |
| Preview | Seconds cell in countdown | **REMOVE** | — |
| Preview | `:` separator between countdown cells | **ADD** | — |

### On `socialProofAvatars`
Up to **5** https URLs uploaded through the same media endpoint; the app renders **three** of them as
the small overlapping circles beside the social-proof line (today those are flat lavender dots).

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
- [ ] Banner upload works end to end, and the S3 CORS blocker in §4 is resolved.
- [ ] Colour swatches reject anything that is not `#RRGGBB`, warn on low contrast, and can be cleared
      back to default.
- [ ] The preview shows no seconds cell and uses `:` between countdown cells.
