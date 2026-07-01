# SME — Usage Analytics & DAU API

Dedicated read-only endpoints for the SME portal to render **Daily Active Users** and
**per-user / per-feature engagement**. Server-to-server, same model as the rest of the SME
surface.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `https://app.stanzasoft.ai`)
**Auth:** `x-api-key: <API_KEY_SECRET>` on every request · **Swagger:** `/api/docs` (group **SME**)
**All endpoints** accept `?days=30` (default 30, max 120) and optional `?startDate=&endDate=` (ISO).
All day-bucketing is **IST** (UTC+5:30). Counts are integers; rates are percentages.

---

## 0. Two data planes (read this first)

| | **Plane A — app tables (live)** | **Plane B — `api_usage` capture** |
|---|---|---|
| Source | the app's own tables (simulation, MCQ, psychometric, doc-reading, AI-content, planner, signups, devices, notifications) | a new per-request capture table written by a global interceptor |
| History | **full, immediately** (read live each call; never purged) | **deploy-day forward only**; raw kept 30d, rollup kept 120d, then purged |
| Powers | engaged-DAU, per-feature usage, signups, retention, premium-vs-free, content, platforms, notifications | true app-wide DAU, per-endpoint usage, heatmap, paywall (402) hits |

**DAU is shown as two labeled series:** `engagedDau` (Plane A — a user who did a *persisted action*; full history, a conservative floor) and `activeDau` (Plane B — any authenticated request; deploy-forward, `null` for pre-deploy days).

**Not captured (intentional, v1):** chat / mains-eval / pyq-variation run **client → Dify directly** (our server never sees them), so they don't appear in `/features`. Capturing them later is a separate Dify decision.

---

## 1. Core

### `GET /sme/analytics/summary`
`{ engagedDau24h, engagedWau7d, engagedMau30d, activeDau24h, activeMau30d, stickiness }` — `stickiness = engagedDau24h / engagedMau30d`.

### `GET /sme/analytics/dau?days=30`
`[{ day, engagedDau, activeDau }]` — `activeDau` is `null` for days before the capture was deployed.

### `GET /sme/analytics/features?days=30`
`[{ feature, users, actions }]` — features: `simulation, mcq, doc_reading, psychometric, ai_flashcard, ai_mnemonic, ai_revision, planner`. (`planner` counts only genuine user-created tasks; auto-seeded rows are excluded.)

### `GET /sme/analytics/users?days=30&page=1&limit=20&search=&sort=lastSeen`
Paginated `{ data, total, page, limit, hasMore }`; rows `{ id, email, name, phoneNumber, premiumExpiresAt, lastSeen, actions30d, isPremium }`. `sort ∈ lastSeen|actions`; `search` matches email/name.

---

## 2. Growth & Retention

### `GET /sme/analytics/signups?days=30` → `[{ day, newUsers }]`
### `GET /sme/analytics/retention?days=30`
`{ cohortSize, d1, d7, d30, d1Pct, d7Pct, d30Pct }` — of users who signed up in the window, how many were active ≥1/7/30 days after signup.
### `GET /sme/analytics/churn-risk?inactiveDays=14&page=1&limit=20`
Paginated; users active within 90 days but **silent for ≥`inactiveDays`**. Rows `{ id, email, name, phoneNumber, lastActivity }`.

---

## 3. Monetization × Engagement

### `GET /sme/analytics/premium-engagement?days=30`
`[{ tier, users, activeUsers, totalActions, avgActionsPerActiveUser }]` for `tier ∈ premium|free` (premium = `premiumExpiresAt > now`).
### `GET /sme/analytics/paywall-hits?days=30`
`[{ route, hits, users }]` — HTTP **402** hits = upgrade-intent. ⚠️ Sourced from raw `api_usage` → limited to the **~30-day** raw window.

---

## 4. Behavioural

### `GET /sme/analytics/heatmap?days=30` → `[{ weekday, hour, count }]` (IST; ~30d raw window)
### `GET /sme/analytics/endpoints?days=30` → `[{ route, calls, users }]` (top 100, from rollup)
### `GET /sme/analytics/platforms` → `{ ios, android, web }` (active devices)
### `GET /sme/analytics/content?days=30` → `{ topDocuments:[{title,subject,readers}], simulation:{started,submitted,completionRate} }`
### `GET /sme/analytics/notifications?days=30` → `{ sent, read, readRate }`

---

## 5. How it's stored & bounded (the "no time-bomb" guarantee)

- **Capture** (`ApiUsageInterceptor`): one row per *authenticated* request — `userId, method, route TEMPLATE (no IDs/query/body/PII), status, durationMs, createdAt`. Batched + fire-and-forget, so it adds ~zero request latency. Public / `x-api-key` / webhook routes (no user) are skipped.
- **Rollup + purge** (BullMQ repeatable jobs, mirroring `payment-reconcile`): nightly `rollup` collapses raw rows into `api_usage_daily` (`day×user×route×count`); `purge` drops raw >30d and rollup >120d. So the tables are permanently bounded.
- **Plane A** reads the app's live tables directly — nothing to maintain or purge there.

## 6. Caveats
- `engagedDau` is a floor (browse-only sessions aren't counted); `activeDau` is the broader "any request" number once the capture has data.
- `paywall-hits` and `heatmap` are raw-sourced → ~30-day window; everything else honours the full `days`/date range (Plane A: months; Plane B rollup: 120d).
- IST always (`days` boundaries resolve in IST).

Related: [[reference_usage_dau_sources]], [[sme-portal-integration]], [[feedback_ist_timezone]].
