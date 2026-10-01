# SME Analytics — Frontend Build Guide

> Changed 2026-09-21 — stale-text pass (telemetry live in 2.0, version ladder, links). Exam-dimension changes follow in [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md).
> Changed 2026-09-21 — ?exam= on every analytics route, examCoverage, purchase-funnel (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

> ## ⛔ BEFORE YOU WRITE ANY UI CODE — INVOKE THE `frontend-design` SKILL
> If you are a Claude/coding session building this dashboard, your **first action** is to
> invoke the **`frontend-design`** skill (`/frontend-design`). Do this **even if no one
> asked you to** and even if the surrounding task didn't mention design. Every screen,
> chart, table, and empty state described below must be designed through that skill — do
> not reach for a default dashboard template. This is a hard requirement, not a suggestion.
>
> Design the dashboard for what it actually is: **an internal operations console for the
> PrepMonkey team to watch how aspirants use the app** — DAU trends, feature pull,
> who's slipping away, and what converts. Audience = operators/founders, not consumers.
> The job of the screen is *fast situational awareness*, then *drill-down*.
>
> The app has been **multi-exam since 2026-08-25** (UPSC CSE, APPSC Group 1, TPSC…), so the
> console is not a UPSC console. Every panel below is now filterable by exam, and the exam
> selector is a first-class part of the design, not a settings toggle — see "The second
> concept" below.

This document is the **API contract + UX intent** for the SME portal's analytics section.
It states exactly what each endpoint returns and what each piece of data is *for*, so you
can design the right visualization rather than guessing.

---

## Connection & conventions

- **Base URL:** `https://app.stanzasoft.ai/api/v1`
- **Auth:** header `x-api-key: <API_KEY_SECRET>` on **every** request. Missing/invalid → `401 { "message": "Invalid or missing API key" }`.
- **Time window:** every endpoint accepts `?days=30` (default `30`, max `120`) and optional `?startDate=YYYY-MM-DD&endDate=YYYY-MM-DD` (ISO; overrides `days`).
- **Exam:** every endpoint accepts an optional `?exam=<slug>`. Omit it → the response is **byte-identical** to the old one. Supply an unknown slug → **`400`**. Supply a known one → some routes **change shape**. See "The second concept" below; the full per-route contract is [SME_USAGE_ANALYTICS.md §0b](./SME_USAGE_ANALYTICS.md).
- **Timezone:** all day buckets are **IST**. A "day" is an IST calendar date.
- **Counts** are integers; **rates** (`*Rate`, `*Pct`) are percentages (0–100); `stickiness` is a 0–1 ratio.
- **Money/PII:** none here. This surface is counts and timestamps only.

### The one concept that shapes the whole UI: two DAU series
DAU comes back as **two lines**, and they mean different things — label them, don't merge them silently:
- **`engagedDau`** — users who did a *real action* (took a sim, answered MCQs, generated a flashcard…). Available for **full history**. A conservative floor.
- **`activeDau`** — users who made *any* authenticated request. Only exists **from the day request-capture was deployed** → it is `null` for earlier days. Render the null region honestly (e.g. a "tracking started" marker), never as zero.

### The second concept: one global exam selector

**Build one selector, at the top of the page, that scopes the whole console.** Not a per-panel
dropdown — operators compare panels against each other, and a screen where three cards are on
different exams is worse than no filter at all.

* **Options:** `All exams` (the default, which sends **no `exam` param** at all) plus one entry
  per exam from **`GET /sme/exams`**. Do not hardcode the list; exams are added from the SME
  portal and a hardcoded list goes stale silently. Label with the exam's display name, send its
  slug.
* **Default to `All exams`** and make that state visually obvious. It is the *exact* number;
  every per-exam view is an approximation of one kind or another (below).
* Put the selection in the URL so a link to "APPSC, last 14 days" is shareable.
* An unknown slug is a **400**, not an empty chart — that is deliberate, so the selector must
  never be able to emit a slug the API does not know. If a 400 does come back, show "that exam
  no longer exists", not an empty dashboard.

#### Three things the client MUST handle

**1. Four routes change shape.** With `?exam=` supplied, `dau`, `paywall-hits`, `heatmap` and
`endpoints` return `{ data, examCoverage }` instead of a bare array. Everything else keeps its
shape. Normalise once, at the fetch layer:

```ts
const rows = Array.isArray(res) ? res : res.data;
const coverage = Array.isArray(res) ? null : res.examCoverage;
```

Do **not** write `res.map(…)` anywhere downstream of a route in that list. `summary` and
`release-health` are the odd ones out: they are objects already, so they gain `examCoverage` as
a **top-level key** rather than a wrapper.

**2. The "since &lt;date&gt;" label is mandatory wherever `examCoverage` appears.**

<!-- captured from staging 2026-09-21, backend f6329e6 -->
Both states, captured minutes apart on 2026-09-21 against `exam=appsc-group-1`:

```json
// GET /sme/analytics/heatmap?days=3&exam=appsc-group-1   — raw api_usage, already stamped
{ "data": [], "examCoverage": { "since": "2026-09-21T14:30:34.917Z", "unknownRows": 64 } }

// GET /sme/analytics/dau?days=3&exam=appsc-group-1       — api_usage_daily rollup, nothing yet
{ "data": [], "examCoverage": { "since": null, "unknownRows": 0 } }

// GET /sme/analytics/paywall-hits?days=30&exam=appsc-group-1
{ "data": [], "examCoverage": { "since": null, "unknownRows": 0 } }

// and WITHOUT ?exam= the same route is still a bare array, unwrapped:
// GET /sme/analytics/paywall-hits?days=30  →  []
```

> **`since: null` with `unknownRows: 0` is the state you will actually ship against on
> day one**, and it is the one most likely to be mis-rendered. It means *this source has
> no exam-stamped row in this window* — not "zero usage", and not "nothing was
> discarded". The two routes above disagree for the same window because one reads raw
> `api_usage` and the other the nightly `api_usage_daily` rollup; **normalise on
> `since === null`, not on `unknownRows > 0`.**

The per-request exam column only exists from the **2026-09-21** deploy. Rows captured before it
are **excluded** — they belong to *no* exam, not to UPSC. So a per-exam usage chart starts on
deploy day and the days before it are **unknown, not zero**.

* Render a persistent chip on the panel: **"Per-exam data from 21 Sep 2026"** (`since`, formatted;
  it is an **ISO instant** on `paywall-hits` / `heatmap` / `summary` / `release-health` and an
  **IST `YYYY-MM-DD`** on `dau` / `endpoints` — format both, do not assume one).
* `since: null` means **not one row in this window carries an exam**. Render the panel as
  *"not recorded for this exam yet"* — never as a zero, never as a flat line at the bottom.
* Show `unknownRows` behind the chip ("64 earlier requests have no exam recorded" — 64 is what
  the `heatmap` capture above returned; each route reports its **own** discard count for its
  **own** window, so never reuse one route's figure on another panel).
* **Clip the series** so the chart starts at `since`, or annotate the boundary. A line that
  falls off a cliff on 20 Sep is the single most likely wrong conclusion this feature can cause.
* **Never put a per-exam number beside a pre-deploy all-exam number.** If a comparison, delta or
  sparkline would straddle `since`, suppress it.

**3. Two different kinds of approximation, and they need different copy.**

| | What it means | Where | Copy |
|---|---|---|---|
| **Retro-attribution** | A person — and *everything they ever did* — is attributed to the exam they have picked **today**. There is no switcher history. | `features`, `users`, `signups`, `retention`, `churn-risk`, `platforms`, `notifications`, `notification-effectiveness`, `onboarding-funnel`, and the *engaged* half of `summary`/`dau` | "Attributed to each user's **current** exam" |
| **Coverage floor** | Rows with no recorded exam are excluded, so the number starts on deploy day. | anything with `examCoverage` | "Per-exam data from &lt;since&gt;" |

Also note: `content` filters on the **content's** tags (a `*`-tagged document counts for every
exam), so per-exam figures there can sum **above** the all-exams total. Do not build a
"100% stacked by exam" chart out of it.

---

## Endpoints

Grouped the way the dashboard should be grouped. Each entry: what it's for → request → **exact response** → field meanings → suggested visualization.

### ① Core — "is the app alive and growing?"

#### `GET /sme/analytics/summary`
The top-of-page headline numbers. One call powers the stat row.
```json
{
  "engagedDau24h": 33,
  "engagedWau7d": 210,
  "engagedMau30d": 540,
  "activeDau24h": 96,
  "activeMau30d": 870,
  "stickiness": 0.061
}
```
| Field | Meaning |
|---|---|
| `engagedDau24h` / `engagedWau7d` / `engagedMau30d` | distinct users who *acted* in the last 1 / 7 / 30 days |
| `activeDau24h` / `activeMau30d` | distinct users who *made any request* (capture-based) |
| `stickiness` | `engagedDau24h ÷ engagedMau30d` — a 0–1 "how habitual" ratio; show as a % |

With `?exam=` this object gains a **top-level `examCoverage`** (no wrapper). ⚠️ The `engaged*`
trio and the `active*` pair are then scoped on *different* columns — the person's exam vs the
request's exam — so per-exam `activeMau30d` is a floor that starts at `examCoverage.since`
while `engagedMau30d` has full history. Put the "since" chip on the stat row, and do not render
"active" as a percentage of "engaged" while the two cover different spans.

→ **Viz:** a row of stat blocks. Lead with DAU; pair each engaged number with its active counterpart as a subordinate value, not a second giant number. Stickiness as a small gauge or labeled %.

#### `GET /sme/analytics/dau?days=30`
The primary chart of the page.
```json
[
  { "day": "2026-06-30", "engagedDau": 33, "activeDau": 96 },
  { "day": "2026-06-29", "engagedDau": 73, "activeDau": 141 },
  { "day": "2026-06-28", "engagedDau": 47, "activeDau": null }
]
```
Array, **newest first**. `activeDau` is `null` before capture was deployed.
With `?exam=` the body becomes `{ "data": [ …the same rows… ], "examCoverage": { "since": "2026-09-21", "unknownRows": 40112 } }` — note `since` is an **IST date** here. The `activeDau` line then starts at `since`; the `engagedDau` line still has full history, so the two legitimately begin on different days. Annotate that, don't hide it.
→ **Viz:** dual-line time series. Two clearly-distinguished lines (engaged vs active); render the `null` stretch of `activeDau` as a gap with a "tracking started here" annotation. Weekly seasonality is expected — don't smooth it away.

#### `GET /sme/analytics/features?days=30`
What users actually do.
```json
[
  { "feature": "planner",       "users": 480, "actions": 2110 },
  { "feature": "mcq",           "users": 120, "actions": 2100 },
  { "feature": "psychometric",  "users": 752, "actions": 757 },
  { "feature": "doc_reading",   "users": 120, "actions": 207 },
  { "feature": "simulation",    "users": 69,  "actions": 88 },
  { "feature": "ai_mnemonic",   "users": 69,  "actions": 88 },
  { "feature": "ai_flashcard",  "users": 69,  "actions": 87 },
  { "feature": "ai_revision",   "users": 1,   "actions": 1 }
]
```
`feature` is a stable key (`simulation, mcq, doc_reading, psychometric, ai_flashcard, ai_mnemonic, ai_revision, planner`); `users` = distinct users, `actions` = total events.
→ **Viz:** horizontal bars sorted by `actions`, with `users` as a secondary encoding (label or paired bar). **Note for copy:** map keys to human names (`mcq` → "Practice questions", `ai_mnemonic` → "AI mnemonics"). Chat / mains-evaluation / pyq-variation are **intentionally absent** (they bypass our backend) — if you show a feature legend, don't imply they're zero; omit them.

#### `GET /sme/analytics/users?days=30&page=1&limit=20&search=&sort=lastSeen`
The drill-down roster. Paginated envelope.
```json
{
  "data": [
    {
      "id": "0d2f…",
      "email": "aspirant@example.com",
      "name": "Meghana R.",
      "phoneNumber": "+91XX…",
      "premiumExpiresAt": "2027-06-29T10:15:00.000Z",
      "lastSeen": "2026-06-30T09:42:11.000Z",
      "actions30d": 84,
      "isPremium": true
    }
  ],
  "total": 2098, "page": 1, "limit": 20, "hasMore": true
}
```
`sort ∈ lastSeen | actions`; `search` matches email/name. `isPremium` is pre-computed. `lastSeen`/`premiumExpiresAt` may be `null`.
→ **Viz:** a dense, scannable table with sticky header, a premium marker, relative timestamps ("2h ago"), and a search box + sort toggle wired to the params. This is the "look up a specific user" surface.

### ② Growth & retention — "are we keeping them?"

#### `GET /sme/analytics/signups?days=30` → `[{ "day": "2026-06-29", "newUsers": 61 }]`
Newest first. → **Viz:** bars under/beside the DAU line for context.

#### `GET /sme/analytics/retention?days=30`
```json
{ "cohortSize": 1590, "d1": 49, "d7": 120, "d30": 240,
  "d1Pct": 3.1, "d7Pct": 7.5, "d30Pct": 15.1 }
```
Of users who **signed up in the window**, how many came back ≥1 / 7 / 30 days later. → **Viz:** a small D1→D7→D30 retention curve or three labeled rings. Always show `cohortSize` so the percentages are trustworthy.

#### `GET /sme/analytics/churn-risk?inactiveDays=14&page=1&limit=20`
```json
{
  "data": [
    { "id": "…", "email": "…", "name": "…", "phoneNumber": "…",
      "lastActivity": "2026-06-10T07:03:00.000Z" }
  ],
  "total": 134, "page": 1, "limit": 20, "hasMore": true
}
```
Users active within 90 days but **silent ≥`inactiveDays`** — a win-back list. → **Viz:** action-oriented list ("Last seen 20 days ago"), sortable, exportable. Empty state = a *good* outcome; write it as such ("No one's slipping right now.").

### ③ Monetization × engagement — "what converts?"

#### `GET /sme/analytics/premium-engagement?days=30`
```json
[
  { "tier": "free",    "users": 2090, "activeUsers": 510, "totalActions": 7400, "avgActionsPerActiveUser": 14.5 },
  { "tier": "premium", "users": 8,    "activeUsers": 8,   "totalActions": 320,  "avgActionsPerActiveUser": 40.0 }
]
```
Compare how hard premium vs free users use the app. → **Viz:** a two-column comparison (avg actions/active user is the punchline). Premium cohort is tiny today (8) — design so a small-N tier still reads clearly and doesn't look broken.

⚠️ **With `?exam=`, "premium" silently means something else.** Same shape, different definition:
without the param, `tier` splits on the **person-level** `premiumExpiresAt > now` mirror; with
it, on a **live entitlement in that exam**. So an APPSC subscriber who has switched their active
exam to UPSC shows up under UPSC as **`free`** — correct, because entitlements are per-exam, but
guaranteed to be reported as a bug unless the UI says so. Label the tier axis differently in the
two states ("Premium (any purchase)" vs "Entitled to &lt;exam&gt;") rather than swapping the
number under an unchanged label.

#### `GET /sme/analytics/paywall-hits?days=30`
```json
[
  { "route": "/pyq/:id/reveal",  "hits": 412, "users": 180 },
  { "route": "/mains/:id/reveal", "hits": 96, "users": 47 }
]
```
HTTP-402 paywall hits = **upgrade intent**. ⚠️ Capture-sourced → **~30-day window only**; if `days>30`, clamp + tell the user. → **Viz:** ranked bars; treat as a "hot leads" panel.
With `?exam=` → `{ data, examCoverage }`, filtered on the exam of the **request** (the right axis for a refusal).

<!-- captured from staging 2026-09-21, backend f6329e6 -->
Both forms, captured 2026-09-21 (staging has no paywall refusals, so `data` is empty —
the **envelope difference** is what this shows):

```json
// GET /sme/analytics/paywall-hits?days=30                      → a bare ARRAY
[]

// GET /sme/analytics/paywall-hits?days=30&exam=appsc-group-1   → an OBJECT
{ "data": [], "examCoverage": { "since": null, "unknownRows": 0 } }
```

#### `GET /sme/analytics/purchase-funnel?days=30&exam=` — **new**
Where the money stops. The only panel that puts the **client's** view of a purchase next to the
**server's**.

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/analytics/purchase-funnel?exam=appsc-group-1&days=3` — verbatim, with the seven
`caveats` strings collapsed (their full text is in
[`SME_USAGE_ANALYTICS.md`](./SME_USAGE_ANALYTICS.md)):

```json
{
  "exam": "appsc-group-1",
  "days": 3,
  "series": [
    { "day": "2026-09-21", "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
      "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 0, "conversionPct": null },
    { "day": "2026-09-20", "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
      "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 0, "conversionPct": null },
    { "day": "2026-09-19", "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
      "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 0, "conversionPct": null }
  ],
  "totals": {
    "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
    "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 0, "conversionPct": null
  },
  "caveats": ["…", "…", "…", "…", "…", "…", "…"]
}
```

> **That is the day-one empty state, captured.** Three days requested → three days
> returned, newest first, zero-filled, and **`conversionPct: null` everywhere including
> `totals`**. Build the panel against this body first: if it renders "0% conversion"
> anywhere above, it is wrong before a single real event arrives. `caveats` came back
> with **7** entries (6 base + 1 for `exam`).

**A constructed illustration** of the two awkward rows the rendering rules below exist
for — *not* a capture (staging has no paywall traffic), but both are reachable:

```jsonc
{ "day": "2026-09-20", "paywallViewed": 2, "upgradeTapped": 1, "checkoutOpened": 1,
  "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 3, "conversionPct": 150 },
{ "day": "2026-09-19", "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
  "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 1, "conversionPct": null }
```

| Field | Meaning |
|---|---|
| the five step counts | **distinct users** who emitted that event on that IST day — not event rows. A paywall re-renders; counting rows would make the metric a function of how chatty the build is. |
| `ordersPaid` | server truth: `orders` at `PAID`, test orders excluded, by the order's own IST day. |
| `conversionPct` | `ordersPaid ÷ paywallViewed × 100`, one decimal. **`null` on a zero denominator, and it can exceed 100.** |
| `totals` | per-day distinct counts **summed** — a user active on three days counts three times. Say so in a tooltip; it is the only total consistent with the chart. |

→ **Viz:** a **stacked/step column per IST day** for the five client steps, with `ordersPaid` as
a distinct mark (a different fill, or a marker on the column — it is a different kind of fact),
and **`conversionPct` as a line on a secondary axis**. Two rendering rules are not optional:

* **`conversionPct: null` is a gap in the line, never a zero.** A `0` there reads as "we showed
  the paywall and converted nobody"; the truth is *there was no denominator* — usually a day
  before the emitting build existed.
* **`conversionPct > 100` must render, not clamp.** It happens when someone buys without a
  recorded paywall view, and it measures **emitter coverage**, not a broken funnel. Let the axis
  go past 100 and annotate the point.

⚠️ Every step is a **floor**: the five events are client-emitted by app 2.0+ only, so a rising
conversion can mean *adoption*, not a better paywall. And with `?exam=` the steps are attributed
to the user's **current** exam while `ordersPaid` uses the order's real exam — a retro-attributed
denominator with an exact numerator. Render the per-exam funnel with a "directional" badge and
keep the all-exams view as the exact one. The `caveats[]` array says all of this in prose:
surface it on the panel, do not drop it.

### ④ Behavioural — "when & how they use it?"

#### `GET /sme/analytics/heatmap?days=30`
`[{ "weekday": 1, "hour": 20, "count": 540 }]` — `weekday` 0=Sun…6=Sat, `hour` 0–23 **IST**. Sparse (only non-zero cells). ⚠️ ~30-day window. → **Viz:** a 7×24 heatmap. This answers "when do we push notifications / run campaigns." Fill missing cells as zero.
With `?exam=` → `{ data, examCoverage }`. ⚠️ Until `examCoverage.since` is ~30 days old, a per-exam heatmap has only a few days of cells — a sparse grid that reads as "nobody uses this exam". Show the coverage chip *inside* the grid, not under it.

#### `GET /sme/analytics/endpoints?days=30`
`[{ "route": "/tasks/today", "calls": 18400, "users": 920 }]` — top 100 routes by calls. → **Viz:** a "traffic by screen" table; route templates are already ID-stripped, safe to show.
With `?exam=` → `{ data, examCoverage }`, `since` as an **IST date** (rollup grain).

#### `GET /sme/analytics/platforms` → `{ "ios": 410, "android": 1280, "web": 95 }`
Distinct active-device users per platform. → **Viz:** a compact donut or three labeled bars.
With `?exam=` this matches the **token owner's** exam, not the device's — label it "how people studying X reach us", never "installs for X".

#### `GET /sme/analytics/content?days=30`
```json
{
  "topDocuments": [ { "title": "Polity — Fundamental Rights", "subject": "Polity", "readers": 88 } ],
  "simulation": { "started": 88, "submitted": 61, "completionRate": 69.3 }
}
```
Most-read docs + mock-test follow-through. → **Viz:** top-docs list + a single completion-rate stat (a gauge).
⚠️ With `?exam=` this filters on the **content's** tags, not the reader's exam — a document tagged `*` belongs to every exam, so per-exam numbers can sum **above** the all-exams total. Never render this one as a share-of-total.

#### `GET /sme/analytics/notifications?days=30` → `{ "sent": 5400, "read": 1320, "readRate": 24.4 }`
Push effectiveness. → **Viz:** one read-rate stat with sent/read context.

---

## Suggested information architecture

A single scrollable console, top-down by urgency (design the actual look via `frontend-design`):
0. **Scope bar** — the window control and the **exam selector**, pinned. Everything below obeys both.
1. **Pulse row** — `summary` stat blocks.
2. **DAU chart** — `dau` (hero of the page) with `signups` as context.
3. **Feature pull** — `features` bars.
4. **Money** — `premium-engagement` + `paywall-hits` side by side, with the **`purchase-funnel`** panel beneath them (it is the "where does the money stop" answer those two only hint at).
5. **Retention & risk** — `retention` curve + `churn-risk` list.
6. **Behaviour** — `heatmap`, `endpoints`, `platforms`, `content`, `notifications` in a denser grid.
7. **User explorer** — the `users` table, full-width, as the drill-down floor.

## States to design (don't skip these)
- **Loading:** per-card skeletons; the page should not block on the slowest call — fire requests independently.
- **Empty:** several panels can legitimately be empty (no churn risk, no paywall hits, a window that reaches back past a capture start date). Write empties as outcomes or invitations, never errors. `api_usage` capture is **live** — `ApiUsageInterceptor` is registered globally in `app.module.ts` — so `activeDau`/capture-based panels are populated; for a window that opens before their capture start, say "Request tracking started on <date>" with the real date, and render `null`, not a zero.
- **Clamped window:** `heatmap` / `paywall-hits` only have ~30 days; if the user picks 60/90, show the data and a quiet note that those two are limited to 30 days. `purchase-funnel` clamps at **90** (`auth_events` retention) — it echoes the `days` it actually used, so render *that*, not the value you sent.
- **Exam-scoped but uncovered:** an exam is selected and `examCoverage.since` is `null`. This is **not** an empty state and **not** an error — it means no request in this window carries an exam yet. Copy: *"Per-exam request data starts on 21 Sep 2026 — nothing in this window."* Never a zero, never a flat line.
- **Exam-scoped and partially covered:** the common case for weeks after this deploy. Draw only from `since` forward and mark the boundary; do not let the chart imply usage collapsed the day before.
- **Error:** state what failed and offer retry, in the console's voice. A **400** from an analytics route almost always means an unknown `exam` slug — say that specifically, and re-fetch `GET /sme/exams`.

## Coverage contract — render EVERY data point (Definition of Done)

**Requirement:** the portal must *consume and visibly surface every field of every
endpoint below.* This is the acceptance bar — the build is **not done** until each item is
rendered somewhere a user can see it (a chart axis, a stat, a table column, a tooltip, or
a labeled value). Do not silently drop a field because it "didn't fit"; if something is
secondary, put it in a tooltip or detail row — but it must be present. Tick each box.

- [ ] **exam selector** → options from `GET /sme/exams` + an `All exams` default; scopes every panel; state lives in the URL; a `400` is reported as "unknown exam", not as an empty dashboard
- [ ] **examCoverage** (wherever present: `summary`, `dau`, `paywall-hits`, `heatmap`, `endpoints`, `release-health`) → `since` rendered as a **"per-exam data from &lt;date&gt;"** chip on that panel (ISO instant *or* IST `YYYY-MM-DD` — handle both) · `unknownRows` surfaced · `since: null` rendered as "not recorded for this exam yet"
- [ ] **wrapped shape** → the client normalises `{ data, examCoverage }` vs a bare array for `dau` / `paywall-hits` / `heatmap` / `endpoints`; no panel breaks when an exam is selected
- [ ] **summary** → `engagedDau24h` · `engagedWau7d` · `engagedMau30d` · `activeDau24h` · `activeMau30d` · `stickiness` · `examCoverage` (exam-scoped only)
- [ ] **dau[]** → `day` · `engagedDau` · `activeDau` (incl. the `null` region shown honestly) · `examCoverage` (exam-scoped only)
- [ ] **features[]** → `feature` (humanized) · `users` · `actions`
- [ ] **users** → `data[]`: `id` · `email` · `name` · `phoneNumber` · `premiumExpiresAt` · `lastSeen` · `actions30d` · `isPremium`; envelope: `total` · `page` · `limit` · `hasMore` (pagination control)
- [ ] **signups[]** → `day` · `newUsers`
- [ ] **retention** → `cohortSize` · `d1` · `d7` · `d30` · `d1Pct` · `d7Pct` · `d30Pct`
- [ ] **churn-risk** → `data[]`: `id` · `email` · `name` · `phoneNumber` · `lastActivity`; envelope: `total` · `page` · `limit` · `hasMore`
- [ ] **premium-engagement[]** → `tier` · `users` · `activeUsers` · `totalActions` · `avgActionsPerActiveUser` (+ the tier label changes meaning under `?exam=` — see §③)
- [ ] **purchase-funnel** → `exam` · `days` (the clamped one) · `series[]`: `day` · `paywallViewed` · `upgradeTapped` · `checkoutOpened` · `checkoutAbandoned` · `purchaseFailed` · `ordersPaid` · `conversionPct` (**`null` as a gap, >100 unclamped**); `totals` (all seven); `caveats[]` rendered
- [ ] **paywall-hits[]** → `route` · `hits` · `users` (+ `examCoverage` when wrapped)
- [ ] **heatmap[]** → `weekday` · `hour` · `count` (7×24 grid, zero-filled) (+ `examCoverage` when wrapped)
- [ ] **endpoints[]** → `route` · `calls` · `users` (+ `examCoverage` when wrapped)
- [ ] **platforms** → `ios` · `android` · `web`
- [ ] **content** → `topDocuments[]`: `title` · `subject` · `readers`; `simulation`: `started` · `submitted` · `completionRate`
- [ ] **notifications** → `sent` · `read` · `readRate`

If a future endpoint or field is added to `/sme/analytics/*`, treat surfacing it as part of
the same contract. The Swagger group **SME** at `/api/docs` is the source of truth for the
live field list — diff against it before calling the dashboard complete.

## Hard reminders
- **Invoke `frontend-design` before building.** (Yes, again.)
- **Render every field — see the Coverage contract above. No data point left unconsumed.**
- Label the **two DAU series** distinctly; never render `activeDau: null` as `0`.
- **One global exam selector**, defaulting to `All exams` (which sends no param). Never a per-panel one.
- **Never render a `null` as a `0`** anywhere on this surface: `activeDau`, `conversionPct`, `examCoverage.since`, `maturedReadRatePct`. Every one of them means "not recorded", and every one of them has a different, honest sentence.
- **Never compare a per-exam number to a pre-deploy all-exam number.** If a delta would straddle `examCoverage.since`, suppress the delta.
- Humanize feature keys and route templates in copy; keep the keys for logic.
- This is internal tooling for operators — optimize for **scannability and trust** (always show denominators like `cohortSize`, `users`) over decoration.
