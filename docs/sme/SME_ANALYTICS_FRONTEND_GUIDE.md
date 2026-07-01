# SME Analytics — Frontend Build Guide

> ## ⛔ BEFORE YOU WRITE ANY UI CODE — INVOKE THE `frontend-design` SKILL
> If you are a Claude/coding session building this dashboard, your **first action** is to
> invoke the **`frontend-design`** skill (`/frontend-design`). Do this **even if no one
> asked you to** and even if the surrounding task didn't mention design. Every screen,
> chart, table, and empty state described below must be designed through that skill — do
> not reach for a default dashboard template. This is a hard requirement, not a suggestion.
>
> Design the dashboard for what it actually is: **an internal operations console for the
> PrepMonkey team to watch how UPSC aspirants use the app** — DAU trends, feature pull,
> who's slipping away, and what converts. Audience = operators/founders, not consumers.
> The job of the screen is *fast situational awareness*, then *drill-down*.

This document is the **API contract + UX intent** for the SME portal's analytics section.
It states exactly what each endpoint returns and what each piece of data is *for*, so you
can design the right visualization rather than guessing.

---

## Connection & conventions

- **Base URL:** `https://app.stanzasoft.ai/api/v1`
- **Auth:** header `x-api-key: <API_KEY_SECRET>` on **every** request. Missing/invalid → `401 { "message": "Invalid or missing API key" }`.
- **Time window:** every endpoint accepts `?days=30` (default `30`, max `120`) and optional `?startDate=YYYY-MM-DD&endDate=YYYY-MM-DD` (ISO; overrides `days`).
- **Timezone:** all day buckets are **IST**. A "day" is an IST calendar date.
- **Counts** are integers; **rates** (`*Rate`, `*Pct`) are percentages (0–100); `stickiness` is a 0–1 ratio.
- **Money/PII:** none here. This surface is counts and timestamps only.

### The one concept that shapes the whole UI: two DAU series
DAU comes back as **two lines**, and they mean different things — label them, don't merge them silently:
- **`engagedDau`** — users who did a *real action* (took a sim, answered MCQs, generated a flashcard…). Available for **full history**. A conservative floor.
- **`activeDau`** — users who made *any* authenticated request. Only exists **from the day request-capture was deployed** → it is `null` for earlier days. Render the null region honestly (e.g. a "tracking started" marker), never as zero.

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
      "phoneNumber": "+9198…",
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

#### `GET /sme/analytics/paywall-hits?days=30`
```json
[
  { "route": "/pyq/:id/reveal",  "hits": 412, "users": 180 },
  { "route": "/mains/:id/reveal", "hits": 96, "users": 47 }
]
```
HTTP-402 paywall hits = **upgrade intent**. ⚠️ Capture-sourced → **~30-day window only**; if `days>30`, clamp + tell the user. → **Viz:** ranked bars; treat as a "hot leads" panel.

### ④ Behavioural — "when & how they use it?"

#### `GET /sme/analytics/heatmap?days=30`
`[{ "weekday": 1, "hour": 20, "count": 540 }]` — `weekday` 0=Sun…6=Sat, `hour` 0–23 **IST**. Sparse (only non-zero cells). ⚠️ ~30-day window. → **Viz:** a 7×24 heatmap. This answers "when do we push notifications / run campaigns." Fill missing cells as zero.

#### `GET /sme/analytics/endpoints?days=30`
`[{ "route": "/tasks/today", "calls": 18400, "users": 920 }]` — top 100 routes by calls. → **Viz:** a "traffic by screen" table; route templates are already ID-stripped, safe to show.

#### `GET /sme/analytics/platforms` → `{ "ios": 410, "android": 1280, "web": 95 }`
Distinct active-device users per platform. → **Viz:** a compact donut or three labeled bars.

#### `GET /sme/analytics/content?days=30`
```json
{
  "topDocuments": [ { "title": "Polity — Fundamental Rights", "subject": "Polity", "readers": 88 } ],
  "simulation": { "started": 88, "submitted": 61, "completionRate": 69.3 }
}
```
Most-read docs + mock-test follow-through. → **Viz:** top-docs list + a single completion-rate stat (a gauge).

#### `GET /sme/analytics/notifications?days=30` → `{ "sent": 5400, "read": 1320, "readRate": 24.4 }`
Push effectiveness. → **Viz:** one read-rate stat with sent/read context.

---

## Suggested information architecture

A single scrollable console, top-down by urgency (design the actual look via `frontend-design`):
1. **Pulse row** — `summary` stat blocks.
2. **DAU chart** — `dau` (hero of the page) with `signups` as context.
3. **Feature pull** — `features` bars.
4. **Money** — `premium-engagement` + `paywall-hits` side by side.
5. **Retention & risk** — `retention` curve + `churn-risk` list.
6. **Behaviour** — `heatmap`, `endpoints`, `platforms`, `content`, `notifications` in a denser grid.
7. **User explorer** — the `users` table, full-width, as the drill-down floor.

## States to design (don't skip these)
- **Loading:** per-card skeletons; the page should not block on the slowest call — fire requests independently.
- **Empty:** several panels can legitimately be empty (no churn risk, no paywall hits, capture not yet deployed). Write empties as outcomes or invitations, never errors. For `activeDau`/capture-based panels before deploy, say "Request tracking starts after the next release" — don't show zeros.
- **Clamped window:** `heatmap` / `paywall-hits` only have ~30 days; if the user picks 60/90, show the data and a quiet note that those two are limited to 30 days.
- **Error:** state what failed and offer retry, in the console's voice.

## Coverage contract — render EVERY data point (Definition of Done)

**Requirement:** the portal must *consume and visibly surface every field of every
endpoint below.* This is the acceptance bar — the build is **not done** until each item is
rendered somewhere a user can see it (a chart axis, a stat, a table column, a tooltip, or
a labeled value). Do not silently drop a field because it "didn't fit"; if something is
secondary, put it in a tooltip or detail row — but it must be present. Tick each box.

- [ ] **summary** → `engagedDau24h` · `engagedWau7d` · `engagedMau30d` · `activeDau24h` · `activeMau30d` · `stickiness`
- [ ] **dau[]** → `day` · `engagedDau` · `activeDau` (incl. the `null` region shown honestly)
- [ ] **features[]** → `feature` (humanized) · `users` · `actions`
- [ ] **users** → `data[]`: `id` · `email` · `name` · `phoneNumber` · `premiumExpiresAt` · `lastSeen` · `actions30d` · `isPremium`; envelope: `total` · `page` · `limit` · `hasMore` (pagination control)
- [ ] **signups[]** → `day` · `newUsers`
- [ ] **retention** → `cohortSize` · `d1` · `d7` · `d30` · `d1Pct` · `d7Pct` · `d30Pct`
- [ ] **churn-risk** → `data[]`: `id` · `email` · `name` · `phoneNumber` · `lastActivity`; envelope: `total` · `page` · `limit` · `hasMore`
- [ ] **premium-engagement[]** → `tier` · `users` · `activeUsers` · `totalActions` · `avgActionsPerActiveUser`
- [ ] **paywall-hits[]** → `route` · `hits` · `users`
- [ ] **heatmap[]** → `weekday` · `hour` · `count` (7×24 grid, zero-filled)
- [ ] **endpoints[]** → `route` · `calls` · `users`
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
- Humanize feature keys and route templates in copy; keep the keys for logic.
- This is internal tooling for operators — optimize for **scannability and trust** (always show denominators like `cohortSize`, `users`) over decoration.
