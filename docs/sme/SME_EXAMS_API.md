# SME Exams API — multi-exam modes

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
    "accessTier": "included",
    "isActive": true,
    "sortOrder": 0,
    "createdAt": "…", "updatedAt": "…"
  }
]
```

### `GET /sme/exams/:id`
One exam. `404` if the slug is unknown.

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
| `accessTier` | — | `included` (default) or `separate`. ⚠️ See §4. |
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

### There is no `DELETE`
Content rows store the slug by value, so deleting an exam would leave rows tagged with
something that no longer exists. **Deactivate instead** — its tagging stays intact for when
it comes back.

---

## 3. Tagging content

No new ingest endpoints. Every existing content-creation call accepts an optional
`examIds` array; **omitting it keeps creating UPSC content, exactly as today.**

| Surface | Field |
|---|---|
| `POST /cms/pyq`, `POST /cms/pyq/bulk` | `examIds` on each item |
| `POST /cms/mains`, `POST /cms/mains/bulk` | `examIds` on each item |
| `POST /cms/simulations` | `examIds` on the simulation (the container, not its questions) |
| `POST /content-doc-admin` | `examIds` |
| `PUT /reels/bulk` | `examIds` on each video |

On **update/upsert**, omitting `examIds` leaves existing tags alone — a PATCH never
silently untags content. Send the field only when you mean to change it.

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

Today **every exam is `included`**: one premium plan unlocks all of them, and the exam
dimension changes nothing about entitlement.

⚠️ **`separate` is NOT IMPLEMENTED.** There is no per-exam purchase, order or entitlement
record anywhere. The flag exists so the decision can be made later by flipping a column
instead of shipping code — but the gate currently **still grants access** when it is set,
deliberately. Failing closed would make that exam's content visible in the switcher and
impossible to buy. Setting it today does nothing except log a warning.

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
2. Load its content, tagging `examIds` on every row.
3. `GET /sme/exams/:id/content-counts` — confirm `exclusive` is non-zero for each module
   you expect. **A zero here means the tags did not land.**
4. `PATCH /sme/exams/:id` with the `enabledModules` you actually have content for.
5. Configure subjects: `GET/PUT /sme/filter-config/prelims?exam=<id>`.
6. `PATCH /sme/exams/:id { "isActive": true }` — it appears in the switcher within ~60s
   (each API replica caches the catalogue for a minute).

A switcher with fewer than two active exams is hidden by the app, so step 6 is the moment
the feature becomes visible to anyone.
