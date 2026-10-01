# SME Exams API — multi-exam modes

> Changed 2026-09-21 — exam-scoping gaps closed (existing contracts).
> Changed 2026-09-21 — exam-safe ingest: /sme/content/* + examIds on /cms and content-doc (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

Managing the list of exams the app can be switched between: **UPSC CSE**, **APPSC Group 1**,
**TPSC Group 2**, and whatever comes next.

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Auth:** `x-api-key: <API_KEY_SECRET>` on every `/sme/*` request.

> **There is no response envelope.** Success bodies are raw; errors are shaped by the
> exception filter. Branch on the HTTP status code, never on a `success` field.

> **Invoke the `frontend-design` skill before building any screen from this doc.**

---

## 1. The whole mental model, in one page

Until now "UPSC" was a constant, not a field. Multi-exam mode adds **one dimension** to
content and **one selector** to the app.

**Every content row carries a list of exam ids** (`examIds`). A row is visible in an exam
if that exam's id is in the list — or if the list contains the wildcard `"*"`, which means
*every exam, including ones that don't exist yet*.

| A row tagged | Is visible in |
|---|---|
| `["upsc-cse"]` | UPSC only |
| `["appsc-group-1"]` | APPSC Group 1 only |
| `["upsc-cse","appsc-group-1"]` | those two, not TPSC |
| `["*"]` | every exam, forever, with no re-tagging when a new one is added |

**Absence always means UPSC.** A request with no exam, a content row created without one,
an unknown or deactivated slug — all resolve to `upsc-cse`. That is deliberate and it is
what lets every already-released app build keep working untouched.

### The one trap worth knowing

Because omitting `examIds` at ingest creates **UPSC** content, an APPSC upload that forgets
the field silently lands in UPSC. Nothing errors. `GET /sme/exams/:id/content-counts`
exists to make that visible — check it after any bulk load.

---

## 2. Endpoints

### `GET /sme/exams`
Every exam, active or not, in display order.

```json
[
  {
    "id": "upsc-cse",
    "displayName": "UPSC Civil Services",
    "shortName": "UPSC",
    "examDate": "2026-09-05T00:00:00.000Z",
    "enabledModules": ["prelims","mains","reels","library","simulation"],
    "accessTier": "paid",
    "appleSubscriptionGroupId": "22017088",
    "logoUrl": null,
    "featureFlags": null,
    "featureCaps": null,
    "trialDays": 14,
    "trialPolicy": [{ "days": 14, "effectiveFrom": "1970-01-01T00:00:00Z" }],
    "paywallContent": null,
    "isActive": true,
    "sortOrder": 0,
    "createdAt": "…", "updatedAt": "…"
  }
]
```

The five per-exam config columns (`logoUrl`, `featureFlags`, `featureCaps`, `trialDays`
/ `trialPolicy`, `paywallContent`) arrived on 2026-09-17 and are covered in §7–§11.
`null` on all of them — which is what UPSC still carries — means "unconfigured", and
unconfigured is byte-for-byte the behaviour that shipped before they existed.

### `GET /sme/exams/:id`
One exam. `404` if the slug is unknown.

The stored row **plus two resolved views**, because the raw override blobs are partial
and usually `null`, and showing an operator `featureFlags: null` tells them nothing about
what the exam actually does:

```json
{
  "id": "appsc-group-1",
  "featureFlags": { "mains": false },
  "featureCaps": { "pyq_reveal": { "type": "daily", "limit": 5 } },
  "trialDays": 0,
  "effectiveFeatureFlags": {
    "prelims": true, "prelims.reveal": true, "…": true,
    "mains": false, "mains.evaluation": false, "mains.modelAnswer": false, "mains.research": false
  },
  "effectiveFeatureCaps": {
    "chat_mentor": { "type": "daily", "limit": 3 },
    "pyq_reveal":  { "type": "daily", "limit": 5 },
    "content_doc": { "type": "lifetime_per_subject", "limit": 1 },
    "pyq_variation": { "type": "premium_only" }
  }
}
```

**Display the effective maps; edit the raw ones.** `effectiveFeatureFlags` carries every
registry key after defaults + overrides + parent-off propagation (note `mains.*` above
went off without anyone listing the children). `effectiveFeatureCaps` is the global
`app_config` limit map with this exam's overrides merged on top.

### `GET /sme/exams/:id/content-counts`
**Check this before switching an exam on.**

```json
{
  "exam": "appsc-group-1",
  "modules": [
    { "module": "prelims",    "visible": 340, "shared": 0,   "exclusive": 340, "enabled": true  },
    { "module": "mains",      "visible": 0,   "shared": 0,   "exclusive": 0,   "enabled": false },
    { "module": "reels",      "visible": 812, "shared": 812, "exclusive": 0,   "enabled": true  },
    { "module": "library",    "visible": 0,   "shared": 0,   "exclusive": 0,   "enabled": false },
    { "module": "simulation", "visible": 0,   "shared": 0,   "exclusive": 0,   "enabled": false }
  ]
}
```

| Field | Means |
|---|---|
| `visible` | Everything this exam's users can see, shared content included. |
| `shared` | Of which is generic content tagged `"*"` that this exam merely inherits. |
| `exclusive` | Authored **for** this exam. **This is the number that says whether the ingest work actually happened.** |
| `enabled` | Whether `enabledModules` currently advertises this module. |

**`exclusive: 0` on a module you just loaded content into is the signature of a forgotten
`examIds`** — those rows became UPSC content. Fix by re-sending them with the right tags.

### `POST /sme/exams`

```json
{
  "id": "appsc-group-1",
  "displayName": "APPSC Group 1",
  "shortName": "APPSC",
  "examDate": "2026-11-15",
  "enabledModules": ["prelims","reels"],
  "isActive": false
}
```

| Field | Required | Notes |
|---|---|---|
| `id` | ✅ | **IMMUTABLE.** Lowercase alphanumeric with single hyphens. Content rows store this slug *by value*, so it can never be renamed — only recreated. |
| `displayName` | ✅ | Full name, shown in the switcher sheet. |
| `shortName` | ✅ | ≤ 24 chars. Renders inline in the app's header chip — keep it short. |
| `examDate` | — | `YYYY-MM-DD`. Drives **this exam's own** countdown, phase and daily-task mix. Omit if unannounced. An impossible date (`2027-02-31`) is rejected, not silently rolled over. |
| `enabledModules` | — | Subset of `prelims`, `mains`, `reels`, `library`, `simulation`. **The app hides anything absent** — this is what lets an exam launch without Mains and with no app release. Defaults to `[]` (nothing shows). |
| `accessTier` | — | `free` (default) or `paid`. ⚠️ See §4 — setting `paid` is REJECTED until the exam has a plan. |
| `appleSubscriptionGroupId` | — | App Store Connect subscription group. ⚠️ Each exam needs its **OWN** group — see §4. |
| `isActive` | — | Defaults `true`. |
| `sortOrder` | — | Ascending order in the switcher. |

**Create it with `isActive: false`.** An inactive exam stays out of the switcher and stops
resolving on user requests, but you can already tag content to it and preview with
`?exam=<id>`. Flip it on when `content-counts` looks right.

`409` if the slug already exists.

### `PATCH /sme/exams/:id`
Any field except `id`. **This is also how you activate and deactivate:**
`{ "isActive": true }`.

`400` if you try to deactivate `upsc-cse` — it is the fallback every un-scoped request and
every already-released app build resolves to, so switching it off would leave those clients
with no content at all.

**The per-exam config fields it also accepts (all added 2026-09-17, all optional, all
MERGED onto what is stored — never a wholesale replace):**

| Field | Shape | Detail |
|---|---|---|
| `featureFlags` | `{ "<dotted key>": boolean }` | Partial. `false` switches a feature off; `true` **removes** the override. §7 |
| `enabledModules` | `string[]` | Still accepted, now a **derived mirror** of the five module flags. §7 |
| `featureCaps` | `{ "<capKey>": {type,limit} \| null }` | Partial. `null` drops that override back to the global limit. §8 |
| `trialDays` | `0`–`90` | **New signups only.** Appends a dated entry to `trialPolicy`. §9 |
| `logoUrl` | `string \| null` | Emblem for the picker and switcher. §10 |
| `paywallContent` | object | SME paywall copy. Sanitised, never rejected. §11 |

A PATCH touching one of these does **not** disturb the others, and a PATCH touching none
of them behaves exactly as it did before they existed.

### There is no `DELETE`
Content rows store the slug by value, so deleting an exam would leave rows tagged with
something that no longer exists. **Deactivate instead** — its tagging stays intact for when
it comes back.

---

## 3. Tagging content

Every content-creation call accepts an `examIds` array. On the legacy `/cms/*` and
`/content-doc-admin` surfaces it is **optional, and omitting it keeps creating UPSC
content, exactly as today.**

**Added 2026-09-21: `/sme/content/*`** — a key-gated mirror of the three question banks
where `examIds` is **required** on every create and every write is audited. Prefer it for
anything new; it is the only way to make the forgotten-tag failure impossible rather than
merely detectable. Full guide: **[SME_CONTENT_INGEST_API.md](./SME_CONTENT_INGEST_API.md)**.

| Surface | Field | Recommended |
|---|---|---|
| `POST /sme/content/pyq`, `POST /sme/content/pyq/bulk` | `examIds` on each item — **required, validated, audited** | ✅ use this |
| `POST /sme/content/mains`, `POST /sme/content/mains/bulk` | `examIds` on each item — **required, validated, audited** | ✅ use this |
| `POST /sme/content/simulations` | `examIds` on the simulation (the container, not its questions) — **required** | ✅ use this |
| `POST /cms/pyq`, `POST /cms/pyq/bulk` | `examIds` on each item — optional, defaults to `["upsc-cse"]` | legacy — existing scripts only |
| `POST /cms/mains`, `POST /cms/mains/bulk` | `examIds` on each item — optional | legacy — existing scripts only |
| `POST /cms/simulations` | `examIds` on the simulation — optional | legacy — existing scripts only |
| `POST /content-doc-admin` | `examIds` — optional, defaults to `["upsc-cse"]` | no keyed mirror exists |
| `PUT /reels/bulk` | `examIds` on each video | |

On **update/upsert**, omitting `examIds` leaves existing tags alone — a PATCH never
silently untags content. Send the field only when you mean to change it. (An *explicitly
empty* `[]` is the one divergence: `/cms/*` reads it as "unsaid" and writes `["upsc-cse"]`;
`/sme/content/*` rejects it with a 400.)

Reading back is scoped too: `?exam=<slug>` now filters `GET /cms/{pyq,mains,simulations}`,
`GET /sme/content/{pyq,mains,simulations}`, `GET /content-doc-admin` and
`GET /sme/content/question-quality`. On all of them an unknown slug is a **400**, never an
empty page that reads as "this exam has no content".

**Recommended defaults**

| Content | Tag as | Why |
|---|---|---|
| Reels / current affairs | `["*"]` | Identical for every Indian competitive exam, and `"*"` means a new exam inherits them with no re-tagging. |
| Library study documents | `["*"]`, or the specific exams | Core GS material is shared; state-specific material is not. |
| PYQ / Mains papers | the one exam | A past paper *is* that exam's paper. |
| Simulations | the one exam | Marking schemes and patterns are exam-specific. |

### Subject filter config is per-exam
`/sme/filter-config/*` gained an optional **`?exam=`** query param, defaulting to
`upsc-cse`. **Every call the portal makes today is unchanged.** Pass `?exam=appsc-group-1`
to edit that exam's subject display names, icons, visibility and order.

⚠️ Unlike the app-facing routes, an unknown `exam` here returns **400**, not a silent
fallback — an admin edit must never quietly redirect itself onto live UPSC config.

⚠️ `POST /sme/filter-config/mains/:subject/optional` flips the rows **visible in that
exam**, which includes rows tagged `"*"`. Flipping a subject that contains shared rows
affects every exam those rows appear in.

---

## 4. Pricing — read this before touching `accessTier`

**Each exam sells its own subscription.** Buying UPSC grants nothing in APPSC and vice versa.
Superseded vocabulary: `included`/`separate` (where `separate` was declared but unimplemented
and deliberately granted access anyway). Both migrated to `paid` on 2026-09-08.

| Value | Meaning |
|---|---|
| `free` (default) | Everyone sees all of this exam's content. No gate, no metering. |
| `paid` | Sold on its own. Access needs a live entitlement **for this exam**, or the trial. |

The free tier inside a `paid` exam starts out the same as UPSC's — browse everything,
3 reveals/day, 1 content doc per subject, explanations withheld until revealed — and is
now tunable per exam (§8).

⚠️ **The trial is one lifetime window per PERSON, but its LENGTH is per exam** (§9). It
was a flat 14 days for everybody until 2026-09-17; `appsc-group-1` was seeded to `0` by
that migration, so an APPSC signup from then on is `free`, not `trial`. **`upsc-cse` was cut
from 14 to 3 days on 2026-09-25** (`PATCH /sme/exams/upsc-cse {"trialDays":3}`, history entry
`effectiveFrom 2026-09-25T04:14:27Z`): UPSC signups from then on get 3 days, everyone who signed
up earlier keeps 14. Read the live value from `GET /sme/exams/:id`, never from this doc.

### ⚠️ `paid` is rejected until the exam has a plan

A `paid` exam with no active plan is content that is **visible but impossible to buy** — the
dead end the old fail-open flag existed to avoid. So the API refuses the flip, and refuses to
retire the last active plan of an exam that is already `paid`. Add plans first (§4.1), then
flip the tier.

### 4.1 `GET|POST /sme/exams/:id/plans`, `DELETE /sme/exams/:id/plans/:planId`

What an exam sells, per billing period. These rows are what let a new exam launch **without an
app release**: the paywall reads `priceInPaise` and `appleProductId` from here and hands them to
the client, so product ids are no longer compiled into the app.

```json
POST /sme/exams/appsc-group-1/plans
{
  "planId": "annual",
  "planType": "ANNUAL",
  "priceInPaise": 590000,
  "strikePriceInPaise": 799900,
  "appleProductId": "com.stanzasoft.upscbuddy.appsc.premium.annual",
  "razorpayPlanId": "plan_XXXXXXXX"
}
```

| Field | Notes |
|---|---|
| `planId` | Public tier slug the paywall renders: `monthly` or `annual`. Upsert key with the exam. |
| `planType` | `MONTHLY` (+30d) or `ANNUAL` (+365d). Drives the grant duration. |
| `priceInPaise` | What **Razorpay** charges and what the **Android/web** paywall shows. ⚠️ iOS renders StoreKit's own price, NOT this — keep it in step with the App Store Connect price tier by hand. |
| `strikePriceInPaise` | **DISPLAY ONLY** (2026-09-17). Rendered struck through beside the real price, with the same period suffix (`₹7,999 /year` above `₹5,900 /year`). Nothing charges it and no grant reads it. Must be **strictly greater** than `priceInPaise` — a strike at or below the real price is an advertised lie, and it is a 400. Omit to leave the stored value alone; send `null` to remove the strike. |
| `appleProductId` | **Globally unique across exams.** Apple webhooks carry only this id, so it is the sole means of knowing which exam a payment bought. Two exams sharing one product would sell a single subscription as two entitlements — the API returns 400 rather than allow it. |
| `razorpayPlanId` | Also globally unique. |

### ⚠️ This upsert MERGES (fixed 2026-09-17)

An omitted field keeps its stored value; only an explicit `null` clears one. It did not
use to. The update branch wrote `appleProductId ?? null`, so the portal's price editor —
which sends only `priceInPaise` — silently **nulled both store product ids**, and a
provider webhook carries nothing but that id, so the next renewal would have resolved to
no exam and granted nothing. Same defect class as the campaign `priceInPaise` wipe.

The strike price is validated against whichever price the row will actually end up with,
so `{ "strikePriceInPaise": 799900 }` on its own is still checked against the stored
`priceInPaise`.

`DELETE` **deactivates, never deletes**: orders and entitlements reference these identifiers,
and a renewal webhook for a deleted product id would have no exam to resolve to.

### ⚠️ Every exam needs its OWN Apple subscription group

Within a single App Store Connect subscription group, StoreKit treats a second purchase as an
upgrade and **REPLACES** the first. Sharing one group across exams would cancel a user's UPSC
subscription the moment they bought APPSC. This is a StoreKit constraint, not a preference —
create a new group per exam and record it in `appleSubscriptionGroupId`.

---

## 5. Push notifications

`POST /sme/notifications/segment` gained an optional **`exam`** field:

```json
{ "segment": "free", "exam": "appsc-group-1", "title": "…", "body": "…" }
```

Omit it to reach the whole cohort, exactly as before.

⚠️ **`exam: "upsc-cse"` also matches every user who has never used the switcher.** There is
no onboarding exam step, so an unset preference means UPSC — matching only the literal
value would reach almost nobody.

`POST /sme/notifications/broadcast` is a **topic** blast and cannot be segmented by exam.

---

## 6. Launch checklist for a new exam

1. `POST /sme/exams` with `isActive: false`.
2. Load its content, tagging `examIds` on every row. **Load it through
   [`/sme/content/*`](./SME_CONTENT_INGEST_API.md)** — there `examIds` is required, so
   forgetting it is a 400 before anything is written rather than a silent UPSC upload you
   discover at step 3. A bad slug is a 400 there too. (Library documents still go through
   `/content-doc-admin`, which has no keyed mirror — tag those by hand and check step 3
   carefully.)
3. `GET /sme/exams/:id/content-counts` — confirm `exclusive` is non-zero for each module
   you expect. **A zero here means the tags did not land.**
4. `PATCH /sme/exams/:id { "featureFlags": { … } }` — switch **off** every feature this exam
   has no content for (§7 has the registry and the merge rules).
   ⚠️ **Only leave a module on if it has content.** APPSC advertised Mains and Reels tabs while
   holding zero rows in each — anyone switching to it opened two empty screens.
   `PATCH { "enabledModules": [...] }` is still accepted as a **legacy alias** and is translated
   into the same flag overrides, but it can only express five coarse module toggles and the
   column is a derived mirror (§7). Write `featureFlags`.

### Launching it as a PAID exam — steps 5a–5e

An exam can launch `free` to build an audience and be flipped to `paid` later; that is a data
change, not a release. When you do want to sell it, do these **between steps 4 and 6**:

- **5a. App Store Connect** — create a **new subscription group for this exam** (never reuse
  another exam's — see §4), add the monthly/annual products, and submit them for review.
  Record the group id in `appleSubscriptionGroupId`.
- **5b. Razorpay** — create the matching plans.
- **5c.** `POST /sme/exams/:id/plans` for each tier, with the real `appleProductId` /
  `razorpayPlanId` and a `priceInPaise` that matches the ASC price tier.
- **5d.** `PATCH /sme/exams/:id { "accessTier": "paid" }` — this is rejected until 5c exists.
- **5e.** Confirm `GET /paywall` with `X-Exam: <id>` returns your plans and not
  `NO_PLANS_AVAILABLE`.

### Then, for every exam — steps 6–7

6. Configure subjects: `GET/PUT /sme/filter-config/prelims?exam=<id>`.
7. `PATCH /sme/exams/:id { "isActive": true }` — it appears in the switcher within ~60s
   (each API replica caches the catalogue for a minute).

A switcher with fewer than two active exams is hidden by the app, so step 7 is the moment
the feature becomes visible to anyone.

Before that last step, decide the exam's **trial length** (§9) and its **free-tier limits**
(§8): both apply to new signups the moment the exam goes live, and `trialDays` in
particular can never be applied retroactively.

---

## 7. Feature flags — what an exam has, per feature

Added 2026-09-17. `enabledModules` could only ever say "this exam has no Mains". Flags say
"this exam has Mains, but not model answers", per route, **server-enforced**, with no deploy.

### The registry is code-defined

```
GET /sme/exams/feature-registry
```

> ⚠️ Note the path: it lives **under `/sme/exams`**, declared above `:id` so the literal
> segment wins the match. Full URL:
> `https://app.stanzasoft.ai/api/v1/sme/exams/feature-registry`.

```json
{
  "features": [
    { "key": "prelims",        "label": "PYQ (Prelims)",        "parent": null,      "quota": null,         "alwaysOn": false, "legacyModule": "prelims" },
    { "key": "prelims.reveal", "label": "Answer explanations",  "parent": "prelims", "quota": "pyq_reveal", "alwaysOn": false, "legacyModule": null },
    { "key": "library.simulation", "label": "Simulation",       "parent": "library", "quota": null,         "alwaysOn": false, "legacyModule": "simulation" },
    { "key": "dashboard",      "label": "Dashboard",            "parent": null,      "quota": null,         "alwaysOn": true,  "legacyModule": null }
  ]
}
```

**Render the toggle list from this endpoint, never from a hardcoded key list.** A key that
appears here is a key the server actually enforces; a key that does not is a switch that
would do nothing. The order is declaration order, so parents always precede their children.

| Entry field | Means |
|---|---|
| `key` | The dotted path you send in `PATCH { featureFlags }`. |
| `label` | Human copy for the toggle. Never shown in the app. |
| `parent` | `null` for a module, otherwise the dotted parent key. |
| `quota` | The freemium cap key this feature is metered by (§8), or `null`. |
| `alwaysOn` | Cannot be switched off. Only `dashboard` carries it. |
| `legacyModule` | Its name in the old `enabledModules` array, or `null`. |

Seven modules, 33 leaves today (41 keys in all, counting the modules and
`dashboard.explore`):
`prelims` (reveal, **explanation**, **approach**, variation, bookmark, followUpChat,
weakTopics, mockTest) · `mains` (evaluation, modelAnswer, **approach**, research) · `reels`
(readMore, bookmark, share, calendar) · `library` (content, collections, simulation) ·
`chat` (mentor, sme, flashcard, mnemonic) · `psychometric` (onboarding, retake) ·
`dashboard` (streak, dailyTasks, countdown, banners, explore → journey, flashcards, reels,
mnemonics).

> **Moved 2026-09-16: `library.mockTest` → `prelims.mockTest`.** Mock tests moved out of
> the Library tab and into Practise in the app, so the flag moved with them. Use
> **`prelims.mockTest`**. The old key is **deleted from the registry**, not kept as an
> alias — it had exactly one reader (`GET /pyq/mock-test`) and no exam had ever stored an
> override for it, and a key that gates nothing is precisely what this registry exists to
> prevent. Keeping it would also have stayed wrong: `appsc-group-1` has `library: false`,
> so parent-off ⇒ children-off was switching mock tests off for an exam that offers them.
>
> `PATCH { "featureFlags": { "library.mockTest": false } }` is now a **400** (unknown key),
> and a stale value hand-written into `feature_flags` is ignored with a warning rather
> than throwing. The total stays **41 keys** — one key moved, none added.

### Content-section flags: hiding TEXT, not selling access

`prelims.explanation`, `prelims.approach` and `mains.approach` are a different kind of
switch from everything else on the list, and they are the only three of their kind. They do
not gate a route and they do not meter anything — they decide whether the server sends the
**answer-explanation text** and the **approach / solve-tip text** for this exam at all. When
one is off the field comes back `null` on every route that would otherwise carry it
(`GET /pyq/:id/details`, `GET /pyq/:id/reveal`, `POST /pyq/:id/submit`, the practice list,
`GET /pyq/mock-test`; `GET /mains/:id`, `GET /mains/:id/reveal` and the mains list for
`mains.approach`), **withheld server-side for every tier, premium included** — because a
paying user staring at an empty "Approach" heading is the complaint these exist to remove.
The typical use is an exam whose content does not have that text written yet: APPSC's
prelims questions carry no solve tip. They have **no effect on metering, on entitlement, or
on the paywall**: a reveal still spends a credit, `canViewExplanation` and `unlockedToday`
are unchanged (they describe what the user has earned, not how much text the exam has), and
because these keys carry no `quota` link they never add or remove a row on the generated
free-vs-premium comparison table (§8). Turning off `prelims.reveal` instead is a different
decision entirely — that takes the *feature* away and 403s the route.

> ⚠️ **These two routes cannot be demonstrated on staging, and the reason is worth
> knowing.** `GET /pyq/:id/details` and `GET /pyq/:id/reveal` both touch Neo4j, and
> staging's Neo4j is unreachable — captured 2026-09-21 (backend `f6329e6`), with a live
> app JWT:
>
> ```json
> {
>   "success": false,
>   "message": "Neo4j is unavailable (driver not initialised)",
>   "error": "Error",
>   "statusCode": 500,
>   "timestamp": "2026-09-21T14:58:25.752Z",
>   "path": "/api/v1/pyq/26ce247f-6033-4cdd-ac3d-35ee29e8805c/details",
>   "method": "GET"
> }
> ```
>
> `…/reveal` returns the identical body. **So the flag behaviour described above — the
> `null`-ing of `answerExplanation` / `solveTip` per exam — is verified against the code
> only; it could not be exercised end-to-end on staging.** The `/sme/*` surfaces in this
> handoff do not depend on Neo4j and were all captured live; the one place it shows up in
> an SME response is `snapshot.streak`, which degrades to `available: false` with this
> same reason string (see [`SME_ACTIVITY_TRAIL_API.md` §1](./SME_ACTIVITY_TRAIL_API.md)).

### `psychometric`: switching the personality test off for an exam

`psychometric` turns off the psychometric test for one exam — APPSC is the first to use
it. `PATCH /sme/exams/appsc-group-1 { "featureFlags": { "psychometric": false } }` takes
both children with it and 403s (`FEATURE_DISABLED`) every route on the user-facing
psychometric controller: `GET /psychometric/test-sets`,
`GET /psychometric/test-sets/:id/questions`, `POST /psychometric/results` and
`GET /psychometric/results`. The **admin/authoring** routes are deliberately not gated —
an exam that does not offer the test must not stop an SME from editing test sets. The
two children split it finer: `psychometric.onboarding` is **client-advisory** (whether
the app shows the psychometric step in its onboarding flow — it is not enforced
server-side and cannot be, because onboarding and a retake hit the identical routes),
while `psychometric.retake` is enforced on `GET /psychometric/results` alone, so an exam
can keep the test but drop the "view my results / retake" screen. Crucially,
**`POST /user/profile/onboarding/complete` is never gated by this flag**: it carries no
`@Feature`, accepts `psychometricTestSkipped: true` with a null
`psychometricTestResultId`, and never reads that id — so a client that skips the step
because the flag is off still finishes onboarding and still lands on `status = ACTIVE`.
Nothing here carries a `quota` link, so switching psychometric off **cannot** add or
remove a row on that exam's generated free-vs-premium comparison table (§8).

### Setting them

```json
PATCH /sme/exams/appsc-group-1
{ "featureFlags": { "mains": false, "reels.readMore": false } }
```

- **Every key defaults to ON.** The column stores only the `false`s, which is why an
  untouched exam (`featureFlags: null`) behaves exactly as it did before flags existed.
- **Partial and MERGED.** Keys you omit keep their current value. A PATCH of
  `reels.readMore` alone cannot silently re-enable `mains`.
- **`true` REMOVES the override** rather than storing it — back to the registry default,
  which is on. `{ "mains": true }` is how you switch Mains back on.
- **Parent off ⇒ children off**, automatically. Sending `{"mains": false}` resolves
  `mains.evaluation`, `mains.modelAnswer` and `mains.research` to false too; do not list
  them. The reverse does not hold — a child can be off under a live parent.
- **`dashboard` is rejected** with a 400: an exam whose home screen can be switched off is
  an exam with no app. Its individual tiles (`dashboard.banners`, `dashboard.streak`, …)
  *are* togglable — `alwaysOn` is scoped to the node that declares it and is deliberately
  not inherited.
- **An unknown key is a 400**, naming the registry. A non-boolean value is a 400 too.
- Changes reach each API replica within **~60 s** (the catalogue snapshot's TTL). The
  writing replica drops its cache immediately, so the operator sees it at once and the
  others converge.

### `enabledModules` is now a DERIVED MIRROR

Every shipped 1.8/1.9/2.0 client parses `enabledModules`, and so does `content-counts`, so the
column survives — but it is **rewritten from the flags on every flag change**, from the
five module keys (`prelims`, `mains`, `reels`, `library`, and `simulation` ⇔
`library.simulation`).

`PATCH { "enabledModules": [...] }` is still accepted and is translated into flag
overrides: a module absent from the array becomes `{"<module>": false}`, a module present
has its override cleared. Send both in one request and `featureFlags` wins — it is applied
second, on purpose.

**Do not treat `enabledModules` as a second source of truth.** Writing it is a coarse way
of writing five flags, and reading it back tells you only about those five.

The `20260917000000_per_exam_config` migration seeded each exam's `featureFlags` from the
`enabledModules` it already had — `appsc-group-1`'s `["prelims"]` became
`{"mains":false,"reels":false,"library":false,"library.simulation":false}` — so no exam's
behaviour changed on day one, and the first PATCH that rewrites the mirror rewrites it to
what it already said.

---

## 8. Per-exam free-tier limits (`featureCaps`)

The freemium limits used to be one global map for every exam, so "3 reveals a day" was a
platform-wide constant. Now the global map is the base and each exam may override it, cap
by cap.

```json
PATCH /sme/exams/appsc-group-1
{ "featureCaps": { "pyq_reveal": { "type": "daily", "limit": 5 } } }
```

**Valid keys** (anything else is a 400 — a limit nothing meters is a setting that does
nothing):

| Cap key | Global default | Gated by flag |
|---|---|---|
| `chat_mentor` | `daily 3` | `chat.mentor` |
| `chat_sme` | `daily 3` | `chat.sme` |
| `chat` | `daily 3` | — legacy alias for older app builds; no flag |
| `flashcard` | `daily 3` | `chat.flashcard` |
| `mnemonic` | `daily 3` | `chat.mnemonic` |
| `mains_eval` | `daily 3` | `mains.evaluation` |
| `pyq_reveal` | `daily 3` | `prelims.reveal` |
| `pyq_mains_reveal` | `daily 3` | `mains.modelAnswer` |
| `content_doc` | `lifetime_per_subject 1` | `library.content` |
| `pyq_variation` | `premium_only` | `prelims.variation` |

**Each entry is a WHOLE cap**, `{ type, limit? }`, never a patch of one sub-field:

- `type` ∈ `daily` · `lifetime` · `lifetime_per_subject` · `premium_only`.
- `limit` is a non-negative integer and is **required** for every type except
  `premium_only`, where it is **dropped** — nothing meters a premium-only feature, so a
  number there would sit in the JSON doing nothing.
- `null` for a key **drops that override** and falls back to the global value.
- Keys you omit keep their stored override. Merging is per cap key.

### The TYPE is yours to set, not just the limit

The controller's `@UsageCapped({type:'daily'})` is only a **default**. An exam's override
replaces the type as well as the number, so the same route can be metered four different
ways with no deploy:

```json
PATCH /sme/exams/appsc-group-1
{ "featureCaps": { "pyq_reveal": { "type": "lifetime", "limit": 25 } } }
```

…gives APPSC 25 free reveals **ever** while UPSC keeps 3 a day.

| `type` | What the server enforces | What the user sees | Resets |
|---|---|---|---|
| `daily` | `limit` per IST day, per exam. Re-opening the *same* resource the same day is free. | "N left today" | 12 AM IST, every day |
| `lifetime` | `limit` **ever**, per (user, exam, feature). Re-opening the same resource is free **forever**. | "N left" | never |
| `lifetime_per_subject` | `limit` per subject, per exam, forever. Only meaningful for `content_doc`. | "1 per subject" | never |
| `premium_only` | no free allowance at all — 402 `PREMIUM_ONLY` | "Premium only" | n/a |

Which is reflected on the paywall automatically: `daily 3` renders "3/day", `lifetime 25`
renders "**25 total**", `lifetime_per_subject 1` renders "1 per subject", `premium_only`
renders "— / ✓".

⚠️ **`lifetime_per_subject` outside `content_doc`.** It is the only cap whose route knows
how to find a subject. Set it on any other key and the server cannot meter per subject, so
it falls back to a **whole-exam `lifetime` allowance of the same number** and logs a
warning. It will not silently un-meter the feature, but it will not do what the name says
either — use `lifetime` if that is what you meant. On `/quota/begin` (the Dify-direct
features: chat, flashcards, mnemonics, mains evaluation, variations) there is no resource
at all, so it falls back to `daily`.

### ⚠️ Switching a type on a live exam resets the counter

`daily` and `lifetime` are **different Redis keys**. A user who has spent 3 of 3 daily
reveals today and is then moved to `lifetime 25` starts that lifetime allowance at **0**,
not at 3 — and moving back the other way likewise starts today's daily counter at 0. The
same applies to the "already unlocked" record: reveals bought under `daily` live in a set
that expires at midnight, reveals bought under `lifetime` live in one that never does.

Nothing throws and nothing is lost — the old key simply stops being read (the daily one
expires on its own; a lifetime one is still there if you switch back). Expect a **one-off
grant of a fresh allowance** to everyone active at the moment you flip it, and prefer to
flip late at night. Changing a *limit* within the same type has no such effect.

Unlike paywall copy (§11), a malformed cap is a **400, not a silent drop** — a wrong limit
is a functional gate, and quietly ignoring it would leave an operator believing they had
changed something they had not.

`GET /sme/exams/:id` returns `effectiveFeatureCaps`: what this exam actually meters, global
map plus overrides. Show that; edit `featureCaps`.

⚠️ **A cap change also changes the paywall.** The Free-vs-Premium comparison table is
generated from these numbers for any configured exam (§11), so `pyq_reveal: 5` makes the
paywall say "5/day" without anyone editing copy. That is the entire point — the hardcoded
"3/day" strings became a lie the moment a limit differed.

Limits take ~60 s to reach every replica (per-exam cache, dropped immediately on the
writing one).

---

## 9. Trial length per exam (`trialDays`)

```json
PATCH /sme/exams/appsc-group-1
{ "trialDays": 3 }
```

`0`–`90`. **`0` means the exam grants no trial at all** — a new signup there is `free`
immediately.

### Three rules, and all three matter

1. **The clock is per PERSON; only the LENGTH is per exam.** One lifetime window that
   starts at signup. Switching exams can never farm a second trial — that is deliberate
   anti-farming, not an oversight.
2. **A change applies to NEW SIGNUPS ONLY.** Writing `trialDays` **appends** a dated entry
   to `trialPolicy`, an append-only history:

   ```json
   "trialPolicy": [
     { "days": 14, "effectiveFrom": "1970-01-01T00:00:00Z" },
     { "days": 0,  "effectiveFrom": "2026-09-17T05:30:00.000Z" }
   ]
   ```

   A user's length is the entry with the greatest `effectiveFrom` **≤ their signup time**.
   So nobody already inside a 14-day window loses days, and there is nothing to backfill.
   The history is never rewritten; a no-op PATCH (same value as stored) appends nothing.
3. **The SME per-user extension still wins.** `user_auth.trial_ends_at`, set by
   `/sme/users/:id/trial-extension` (see `SME_TRIAL_EXTENSION_API.md`), overrides the
   computed window for that person regardless of the exam's length.

### What the migration already did

`20260917000000_per_exam_config` seeded **`appsc-group-1` to `trialDays: 0`**, with history
`[{14, epoch}, {0, <migration time>}]`. Existing APPSC users keep the window they were
already inside; everyone who signs up after the migration gets none. `upsc-cse` was left
untouched at 14 from the epoch, which reproduces the old compile-time constant exactly.

### Two consequences worth knowing

- **`trial_started` push now fires at onboarding completion**, not at signup — at signup
  the exam has not been chosen yet, so the only length available was UPSC's 14, and a user
  who went on to pick a 0-day exam was welcomed to a trial they did not have. It is
  **skipped entirely when the chosen exam grants 0 days** (the template's whole body is
  "You have full access for the next {{trialDays}} days"). Dedupe is still once per person.
- **The person-level `user_auth.status` mirror uses the MAXIMUM trial across active exams.**
  That column, once written `UNSUBSCRIBED`, ends the trial in every exam at once — so a
  person is on trial while *any* active exam still grants them one. Shortening UPSC to 3
  while another exam still grants 14 therefore does **not** cut the other exam short.
  Inactive exams are excluded.

---

## 10. Exam logo (`logoUrl`)

```json
PATCH /sme/exams/appsc-group-1
{ "logoUrl": "https://prepmonkey-….s3.ap-south-1.amazonaws.com/exam-logos/appsc.png" }
```

Upload it first:

```json
POST /sme/media/upload-url
{ "filename": "appsc.png", "contentType": "image/png", "folder": "exam-logos" }
```

`folder` now accepts `offers` · `banners` · `exam-logos`; `contentType` is
`image/jpeg` · `image/png` · `image/webp`. PUT the bytes to the returned presigned URL
(no auth header — the signature *is* the authorisation, 5-minute expiry), then store the
returned public URL here. Same flow as offer artwork; see `SME_OFFERS_API.md` §7.

**Author it at 512×512 PNG with transparency.** Nothing validates dimensions — they cannot
be checked from a URL without fetching it.

Where it renders: the onboarding exam picker and the home switcher dropdown. Send `null`
(or `""`) to clear it — the client then falls back to the initials of `shortName` and never
renders a blank card. On the app-facing `GET /exams` the field is emitted as `""` rather
than `null`, because an absent/null key is fatal to the KMP client's parser.

---

## 11. Paywall copy per exam (`paywallContent`)

The standard (no-campaign) paywall was hardcoded in three places. Now each exam may author
its own copy, and the Free-vs-Premium table is **generated** from that exam's real limits.

### The exact stored shape

```json
PATCH /sme/exams/appsc-group-1
{
  "paywallContent": {
    "titleLine1": "Crack APPSC",
    "titleLine2": "with Premium.",
    "highlightWord": "APPSC",
    "highlightStyle": "CORAL",
    "benefitsTitle": "Member Benefits",
    "benefits": [
      "Telugu-state focused PYQs",
      "Mentor assisted unlimited chats"
    ],
    "socialProof": "Join 5000+ serious aspirants",
    "ctaLabel": "GO PREMIUM",
    "extraComparisonRows": [
      { "feature": "Doubt clearing calls", "free": "No", "premium": "Weekly" }
    ]
  }
}
```

| Field | Rules |
|---|---|
| `titleLine1` `titleLine2` `highlightWord` `benefitsTitle` `socialProof` `ctaLabel` | Strings, trimmed, truncated at 240 chars. Blank/whitespace is treated as absent. |
| `highlightStyle` | **A STRING ENUM**, one of `NONE` · `TRICOLOR` · `CORAL` · `LAVENDER`. Case-insensitive in, stored uppercase. Anything else is dropped. |
| `benefits` | Array of strings, max 12; blanks removed. `[]` is a real decision ("no bullets") and is honoured. |
| `extraComparisonRows` | Array of `{ feature, free, premium }`, max 12. `label` is accepted as a legacy alias for `feature`, and rows saved under the old spelling still render. A row with an empty `feature` is dropped. |

- **MERGED field by field** with what is stored: a PATCH of `ctaLabel` alone cannot wipe
  the benefits list.
- **SANITISED, never rejected.** A bad value costs one line of default text rather than
  400-ing the whole edit — the same policy as campaign content, and it is what stops an
  older or newer portal build bricking saves. (A bad `highlightStyle` is dropped *and*
  logged.)
- `comparisonTitle` is **not** authorable — the column has no such key.

> ⚠️ `highlightStyle` really is a plain string here, not the `{color,bold}` object the
> campaign `styles` map uses. The sanitiser briefly shared the campaign's object-shaped
> code, and the mobile DTO types this field as a non-nullable `String`, so the first SME to
> set it would have killed the parse on **every** standard paywall — invisibly, because the
> app's offline fallback is a pixel-identical replica. The write side stores the enum and
> the read side accepts nothing else, so a row saved under the old shape renders the
> standard style rather than breaking the screen.

### The comparison table is GENERATED — do not restate limits in it

The Free-vs-Premium table on the standard paywall is built from
**registry leaves × this exam's flags × this exam's caps**:

- a leaf with no `quota` link gets no row (a generated "✓ / ✓" sells nothing);
- a leaf whose flag is **off** is skipped entirely — advertising a limit on a feature the
  exam does not have is a lie;
- the free cell is rendered from the cap: `daily 5` → `5/day`, `lifetime_per_subject 1` →
  `1 per subject`, `lifetime 3` → `3 total`, `premium_only` → `—` (premium cell `✓`);
  everything else → `Unlimited`.

Your `extraComparisonRows` are **appended after** the generated rows. Put *marketing* there
— doubt-clearing calls, mentor sessions — and **never a metered feature's limits**: it is
already in the table, generated from the number the server actually enforces, and a
hand-typed copy of it goes stale the moment §8 changes.

### The legacy path, and why UPSC has not moved

When an exam has **no `paywallContent`, no `featureFlags` and no `featureCaps`** — which is
`upsc-cse` today, and every exam until someone edits it — `/paywall` returns the original
hand-written content **verbatim**, including its twelve hand-written comparison rows. Not
regenerated to look the same: returned. Some of those rows say things the registry cannot
("Psychometric Test", "Personalised dashboard"), so any generated table would differ
visibly on a screen nobody asked to change.

**Writing any one of the three flips that exam onto the generated table.** Editing
`featureCaps` for an exam is therefore also a paywall change; that is intended, but know it
before you do it on UPSC. A change to the *global* `app_config.feature_caps` deliberately
does **not** flip anyone — that column has never fed this screen.

---

## 12. What the app actually enforces

Everything in §7–§9 is enforced **server-side**. The flags on `/config` are advisory for
the UI only; the refusal below is the real gate.

### 403 `FEATURE_DISABLED`

```json
{
  "success": false,
  "message": "Feature \"reels.readMore\" is not available for exam \"appsc-group-1\".",
  "error": "Forbidden",
  "code": "FEATURE_DISABLED",
  "feature": "reels.readMore",
  "exam": "appsc-group-1",
  "statusCode": 403,
  "timestamp": "2026-09-17T…",
  "path": "/api/v1/reels/abc/blog",
  "method": "GET"
}
```

**403, deliberately not 402.** A 402 means "out of credits, buy premium" and opens the
paywall; showing a paywall for a feature the exam does not have would be selling something
that does not exist. Every other 403 in the service (SME api-key, throttling) keeps its
old generic body byte for byte — the passthrough is gated on `code`.

### Which routes are gated

| Registry key | Routes |
|---|---|
| `prelims` | the whole PYQ prelims controller |
| `prelims.reveal` | `GET /pyq/:id/reveal` |
| `prelims.weakTopics` | the weak-topic routes |
| `prelims.bookmark` | PYQ bookmark |
| `prelims.mockTest` | `GET /pyq/mock-test` — **was `library.mockTest`** until 2026-09-16 |
| `mains` | the whole Mains controller |
| `mains.evaluation` | `POST` and `GET /mains/:id/evaluation` |
| `mains.modelAnswer` | `GET /mains/:id/reveal` |
| `reels` | `GET /reels` and the whole reels-user controller |
| `reels.readMore` | `GET /reels/:reelId/blog` |
| `reels.bookmark` | `POST /reels/:reelId/bookmark`, `GET /reels/bookmarks` |
| `library.content` | the content-doc user controller |
| `library.collections` | the user-content (collections) controller |
| `library.simulation` | the simulation controller |
| `chat` | the whole chat controller |
| `psychometric` | the whole user-facing psychometric controller |
| `psychometric.retake` | `GET /psychometric/results` |
| `dashboard.banners` | `GET /banners` |

A handler-level decorator wins over its controller's, which is what lets the reels
controller be `reels` while one route on it is `reels.readMore`.

`prelims.explanation`, `prelims.approach` and `mains.approach` are deliberately **absent
from this table**: they gate no route. They null a field on the way out (§7, "Content-section
flags"), so the request still succeeds and still meters. `psychometric.onboarding` is
absent for a different reason — it is client-advisory only, because onboarding and a
retake are the same routes server-side.

### Daily tasks follow the flags too

`GET /tasks/today` seeds and serves the three engine-backed daily tasks against the
exam's effective flags: `static_reels` needs `reels`, `static_pyq` needs `prelims`,
`static_chat` needs `chat`. A task whose module is off is neither created nor served —
it is an instruction to go and use a module the exam does not have, the client refuses
the deep link, and the streak engine can never auto-complete it, so the day never
closes. Only the module key counts: `prelims.reveal: false` still leaves the PYQ task,
because attempting questions still works. A user's **own** tasks are never filtered, an
unmapped `sourceId` is kept, and an unreadable catalogue serves everything (fail open).
Filtering happens at serve time as well as at seed time, so switching a flag off takes
the task off the dashboard immediately — **no migration deletes the rows already
seeded**. To tidy them up for an exam you have just changed, an SME can run:
`DELETE FROM custom_tasks WHERE source_id = 'static_reels' AND scheduled_date >= CURRENT_DATE;`
(restricted to the affected users). It is optional — they are already hidden.

**SME, CMS and webhook routes are never gated.** An operator must always be able to
configure an exam whose features are off.

### Dify-direct features are refused at `/quota/begin`

Chat, flashcards, mnemonics, mains evaluation and PYQ variations never touch a decorated
controller — the app reserves a credit and then calls Dify itself. So the registry's
`quota` link (§8) is enforced in **`POST /quota/begin`** and in the metering guard instead:
if the flag behind that cap key is off for the exam, the reservation is refused with the
same 403, **before** any credit is spent and before any token is issued.

### Failure semantics

Both gates fail **OPEN**: a decorator carrying a typo, or a catalogue read that errors,
allows the request and logs. A flag that does nothing is far better than a live route
silently 403-ing for everyone.

### The invariant

**`upsc-cse` with no overrides is byte-identical to before all of this**, and a request
carrying no exam header at all resolves to `upsc-cse`. Every flag resolves true, so no
gate ever throws; the caps are the global map unchanged; the trial is 14 days from the
epoch entry; the paywall returns the legacy content verbatim; `enabledModules` derives back
to the same five names. That covers every shipped 1.8/1.9/2.0 install and the whole of
prepmonkey-web. It is checked rather than asserted —
`scripts/verify-exam-backcompat.ts --compare` captures ~30 endpoints before and after a
deploy and fails if any un-scoped response moved.

### Additive keys the app gained

**App 2.0 (released 2026-09-17) is the first build that READS `/config`'s `features` map.**
1.8 and 1.9 ignore it entirely — which is exactly why every flag is also enforced
server-side rather than trusted to the client.

| Endpoint | New key |
|---|---|
| `GET /config` | `features: Record<string, boolean>` — the full effective flag map for the request's exam. Advisory; an unknown key on the client means true. Read by 2.0 and later only. |
| `GET /exams` | `logoUrl` (`""` when unset) and `accessTier` (`free`/`paid`) |
| `GET /exams/:id` | the same two keys — it returns the identical `ExamResponseDto`, so anything true of a row in the list is true of the single fetch. |
| `GET /user/profile/me` | `activeExamId` |
| `POST /user/profile/onboarding/complete` | accepts `examId`, which is what writes `user_profiles.active_exam_id`. Absent ⇒ `upsc-cse`; an unknown slug stores `upsc-cse` rather than failing onboarding. |
| `GET /quota/me`, 402 bodies | `resetsAt` on every `daily` entry (next IST midnight). **Absent for `lifetime`** — those credits never come back, and quoting an instant would promise otherwise. |
| `GET /pyq/:id/details`, `GET /mains/:id` | `unlockedToday` — the resource was already revealed, so it is shown inline and re-reading costs nothing. Under a `lifetime` cap it stays true **forever**, not just for the rest of the day, despite the name. |

### What the client keys on for each type

`GET /quota/me` returns one entry per cap key, and **`type` is the discriminator** — the
client must branch on it rather than assuming `daily`:

| `features[key].type` | Fields present | Counter/sheet copy |
|---|---|---|
| `daily` | `used`, `limit`, `remaining`, `resetsAt` | "N left today", "resets 12:00 AM" |
| `lifetime` | `used`, `limit`, `remaining` — **no `resetsAt`** | "N left" (never "today") |
| `lifetime_per_subject` | `limit`, `perSubject: true` — no `used`/`remaining` | "1 per subject" |
| `premium_only` | `type` only | "Premium only" |

A premium user gets `{unlimited: true, type}` for every key (plus `resetsAt` on `daily`,
informational). The 402 body carries `code`, `feature`, `limit`, `used`, and `resetsAt`
**only** when the cap is `daily`.

⚠️ `active_exam_id` is for push segmentation and cross-device restore **only**. Server-side
request scoping is still `?exam=` → `X-Exam` → `upsc-cse`, and nothing else: reading the
profile column there would move an already-shipped build onto another exam behind its back.
