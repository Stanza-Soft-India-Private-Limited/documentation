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

## 4b. Release & messaging health (see [SME_INSIGHTS_API.md](./SME_INSIGHTS_API.md))

Two further `/sme/analytics/*` endpoints live in their own doc because their caveats
are load-bearing:

### `GET /sme/analytics/release-health?days=14&platform=`
Per **app version × platform**: `activeUsers, requests, errors4xx, errors5xx, errorRatePct,
errorRateExcludingPaywallPct, clientErrors, clientErrorsPerActiveUser, adoptionSharePct`
plus a `comparison` against the next-older build (`verdict ∈ better|worse|similar|insufficient_data`)
and a per-platform rollup with `preHeaderSharePct`. Supplies the number behind an
`AppConfig.minAppVersion` force-update decision.
⚠️ `app_version IS NULL` **is** the pre-1.7 cohort (the header shipped with 1.7) and is
reported as its own labelled bucket — never dropped. ⚠️ `app_version`/`platform`/`status`
exist only on **raw** `api_usage`, so the window is hard-capped at **30 days**.

### `GET /sme/analytics/notification-effectiveness?days=30`
Per notification `type` and per IST day: `sent, read, maturedSent, maturedRead,
maturedReadRatePct, pending, wastedSends, recipients`, plus an explicit
`worstPerformers` kill-list ranked by wasted sends.
⚠️ **`isRead` is a bulk "opened the in-app feed" flag** set by `markFeedSeen`
(`POST /notifications/feed/seen`) — **not** a per-push open, and push delivery is not
tracked at all. ⚠️ **Time-to-read is derivable only from 2026-07-23 onward.** A `read_at` column was
added that day and `markFeedSeen` now stamps it, but rows marked read BEFORE that have
`read_at = NULL`. Never read NULL as "instant" — it means "read before we recorded when". Rates are quoted on a **matured** sample only (default: rows ≥3 days old).

> Note: the older `GET /sme/analytics/notifications` (§4 above) returns the flat
> `{ sent, read, readRate }` with **no maturation split and no isRead caveat**. Prefer
> `notification-effectiveness` for any decision.

---

## 5. How it's stored & bounded (the "no time-bomb" guarantee)

- **Capture** (`ApiUsageInterceptor`): one row per *authenticated* request — `userId, method, route TEMPLATE (no IDs/query/body/PII), status, durationMs, createdAt`. Batched + fire-and-forget, so it adds ~zero request latency. Public / `x-api-key` / webhook routes (no user) are skipped.
- **Rollup + purge** (BullMQ repeatable jobs, mirroring `payment-reconcile`): nightly `rollup` collapses raw rows into `api_usage_daily` (`day×user×route×count`); `purge` drops raw >30d and rollup >120d. So the tables are permanently bounded.
- **Plane A** reads the app's live tables directly — nothing to maintain or purge there.

## 6. Caveats
- `engagedDau` is a floor (browse-only sessions aren't counted); `activeDau` is the broader "any request" number once the capture has data.
- `paywall-hits` and `heatmap` are raw-sourced → ~30-day window; everything else honours the full `days`/date range (Plane A: months; Plane B rollup: 120d).
- IST always (`days` boundaries resolve in IST).

---

## 7. Auth diagnostics + api_usage client context (raw tables)

Two additive data sources landed for auth-failure triage and per-device attribution.
No SME endpoint reads them yet (that's a later module) — query the raw tables directly.

### `auth_events` — pre-auth + auth-lifecycle sink
Written by the **@Public** `POST /diagnostics/auth-events` ingest (mobile emitters).
The whole point is capturing failures *before* a user is authenticated (login / OTP /
refresh), so there is **no FK on `user_id`** (rows survive account deletion). The raw
login identifier is **never stored** — only `identifier_masked` (last-4 visible, e.g.
`********7935`) and `identifier_hash` (SHA-256 hex of the normalized value: email
lowercased, phone digits-only). To look up a phone, hash it the same way and match on
the hash.

`event_type ∈ login_attempt | login_success | login_failed | otp_send_failed | otp_verify_failed | refresh_failed | forced_logout`
(validated at ingest, stored as text for forward-compat). Other columns: `device_id`,
`platform`, `app_version`, `device_model`, `os_version`, `error_code`, `message` (≤500),
`ip`, `created_at`. Rate-limited 60/hour per device (fallback IP) — silent 429.

```sql
-- Failed logins in the last 7 days for a specific phone (hash it the same way the
-- server does: digits-only, then sha256 hex).
SELECT created_at, event_type, error_code, platform, app_version, ip
FROM auth_events
WHERE identifier_hash = encode(digest('919876547935', 'sha256'), 'hex')  -- pgcrypto
  AND event_type IN ('login_failed', 'otp_verify_failed', 'otp_send_failed')
  AND created_at > now() - interval '7 days'
ORDER BY created_at DESC;

-- Which device_ids has one user authed from? (correlate account sharing / churn)
SELECT DISTINCT device_id, max(created_at) AS last_seen
FROM auth_events
WHERE user_id = $1 AND device_id IS NOT NULL
GROUP BY device_id
ORDER BY last_seen DESC;
```

### `api_usage` context columns
`api_usage` gained nullable `app_version`, `platform`, `device_id` (from the
`x-app-version` / `x-platform` / `x-device-id` request headers). Row-level context
only — the `api_usage_daily` rollup is **untouched** (still `day × user × route`).
Populated deploy-day forward for header-sending clients; older rows stay null.

```sql
-- Distinct devices per user from real API traffic (30d raw window).
SELECT user_id, count(DISTINCT device_id) AS devices
FROM api_usage
WHERE device_id IS NOT NULL AND created_at > now() - interval '30 days'
GROUP BY user_id
ORDER BY devices DESC;

-- App-version spread of active users (last 7 days).
SELECT app_version, platform, count(DISTINCT user_id) AS users
FROM api_usage
WHERE app_version IS NOT NULL AND created_at > now() - interval '7 days'
GROUP BY app_version, platform
ORDER BY users DESC;
```

---

Related: [[reference_usage_dau_sources]], [[sme-portal-integration]], [[feedback_ist_timezone]].
