# S02 — `/engagement` (`EngagementAnalytics.tsx`) built from `GET /sme/analytics/{metric}` (Phase 1 verifier)

Read-only pass, 2026-11-03. Portal at `7b00daf` (v2.44.0). Backend `main` = `origin/main` at `4ea16f2` (docs-only on top of `eafb96b`). No SQL was run, no process was started. Every verdict below is code-derived; each row has a reconcile query in §A.3.

Path keys:
- `E:` = `upsc-document-generator/admin-ui/src/pages/EngagementAnalytics.tsx`
- `API:` = `admin-ui/src/services/api.ts`; `INS:` = `admin-ui/src/services/insights.api.ts`; `XS:` = `admin-ui/src/components/features/insights/ExamScope.tsx`
- `M:` = `upsc-document-generator/app/main.py`
- `AC:` = `backend/src/modules/sme/controllers/sme-analytics.controller.ts`; `AS:` = `backend/src/modules/sme/services/sme-analytics.service.ts`; `DTO:` = `backend/src/modules/sme/dto/sme-analytics.dto.ts`
- `RU:` = `backend/src/modules/sme/services/api-usage-rollup.service.ts`; `INT:` = `backend/src/common/interceptors/api-usage.interceptor.ts`; `GO:` = `backend/src/common/guards/global-guard-order.ts`; `UQG:` = `backend/src/common/guards/usage-quota.guard.ts`
- `SUA:` / `DEC:` / `FG:` = `backend/docs/sme/{SME_USAGE_ANALYTICS,SME_METRICS_DECODER,SME_ANALYTICS_FRONTEND_GUIDE}.md`

## A.0 The data path, verified

- **UI → portal.** E:375–401 fires 14 independent queries (plus video, out of scope). Each goes through `fetchApi(..., 60000)`, a 60 s client timeout (API:2792–2807). `uaQuery` drops `undefined`/`null`/`''` (API:2784–2790). The React Query key carries `exam`, `refreshKey` and the per-card window (E:373).
  - Called with `days`: dau, signups, features, premium-engagement, paywall-hits, retention, heatmap, endpoints, content, notifications.
  - Called without `days`: summary and platforms (E:375, 384). churn-risk takes `inactiveDays/page/limit` (E:401). users takes `days/page/limit/search/sort` (E:395).
- **Portal.** The catch-all `GET /sme/analytics/{metric}` (M:6211–6240) is guarded by `require_analytics_access` (admin, app_admin, app_admin_users, analytics, sales; `app/auth.py:416–428`).
  - It allowlists 14 metrics (M:6186–6190); anything else is a 404.
  - It forwards exactly `days, startDate, endDate, page, limit, search, sort, inactiveDays, exam`, and `_sme_proxy` drops `None` values (M:4352–4355).
  - **It drops nothing the UI sends, and it adds nothing.**
  - FastAPI bounds match ours: `days` 1–120 = DTO:28–34; `limit` 1–100 = DTO:56–62. `inactiveDays` has `ge=1` and no upper bound, while ours has `@Max(90)` (DTO:82–88); the UI only offers 7/14/30 (E:646).
  - Routes declared above the catch-all (release-health M:5908, notification-effectiveness M:5924, onboarding-funnel M:5947, video M:5993, ai-assistant M:6130, purchase-funnel M:6193) do not shadow any metric on this page.
- **Bug class 1 (httpx params replacing the query) does not apply.** Every path is built without a query string (M:6233–6239), and `_sme_proxy` merges anyway (M:4357–4366). Bug classes 2, 3 and 7 do not apply either: no `_sme_fetch_all`, no local filtering, explicit `Query()` params.
- **Shapes.** We return raw bodies: no envelope, because `ResponseInterceptor` is never registered (AC:29–34). The portal returns `response.json()` untouched (M:4378). `fetchApi` wraps it as `{success, data}` (API:112–115), and the page unwraps it with `sd(q)` (E:405).
  - The four routes that switch shape under `?exam=` are normalised once, by `unwrapExamRows` (E:409–413, INS:70–77).
  - `summary.examCoverage` is read top-level (E:469).
- **Errors.** Our 400 (unknown slug, from `resolveExam` AS:61–65) becomes a portal `HTTPException(400, detail=message)` (M:4383–4397). `fetchApi` reads `error.detail` (API:107–110), and `isUnknownExamError` (INS:83–86) turns it into `UnknownExamState`, using summary as the page canary (E:430–437). ✔
  - ⚠️ `_sme_proxy` times out at **30 s** (M:4342). The client allows 60 s. A slow full-history query (`summary`/`churn-risk` scan the whole ENGAGED union, AS:135–140, 374–385) surfaces as **503 "Unable to connect to SME management service"** (M:4398–4404), and the card reads "data source temporarily unavailable" (E:100). It is a timeout, not an outage. ⚪
- **Video section (out of scope, except one call).** `_fetch_reels_map` (M:5962–5990) calls our keyless `GET /api/v1/reels?page=N&limit=100` with **no auth header**, up to 10 pages (1,000 reels). It needs `data[].playbackId/id/title/subject` and `meta.hasNext`.
  - Our side is `reels.controller.ts` (class `@Public()`, `getAllReels` → `findAllReels(filters)` with **no exam**, so all exams). Pagination `limit` max is 100 (`pagination.dto.ts:26`), and `meta.hasNext = page < totalPages` (`pagination.util.ts:37`). Contract compatible.
  - One nuance: `buildReelsWhere` hides future-dated reels (`reels.service.ts:316–322`), so a scheduled reel that somehow has Mux views would render as "(deleted reel)" (E:853). ⚪ Mux numbers were not audited.
- **Capture plane facts that decide several verdicts below.**
  1. `api_usage` is written by `ApiUsageInterceptor`, the sole `APP_INTERCEPTOR` (`app.module.ts:143–146`). It records success and thrown errors (INT:120–125), but only for requests that **reach the interceptor**.
     - All five global guards run **before** interceptors (GO:28–34, Nest lifecycle).
     - So any request refused by a guard is never captured: the `UsageQuotaGuard` 402s (UQG:150, UQG:322–334), `FeatureGuard` 403s, and `JwtAuthGuard` 401s. See #11.
  2. `route` is stored **with the global prefix**, e.g. `/api/v1/pyq/:id/reveal` (INT:137–141; prefix from `main.ts:277`). The schema comment `/pyq/:id/reveal` (`schema.prisma:721`) is stale.
  3. Rollup (RU:95–117) runs at 00:30 IST (`0 19 * * *` UTC, RU:16) over `created_at >= now() - 2 days` and does `ON CONFLICT … DO UPDATE SET count = EXCLUDED.count`. Consequences:
     - (a) The row for **today** is written at 00:30 IST from 30 minutes of traffic, and is not completed until the next night.
     - (b) Each day's final count is written by the run two nights later. That run only sees the day from **00:30 IST**, and it *overwrites* the full count. The requests made at 00:00–00:30 IST are therefore lost from `count` for every (user, route) pair that was also active later that day. Distinct-user counts are unaffected.
  4. Purge (RU:124–141) deletes raw `api_usage` older than 30 days (`RAW_RETENTION_DAYS = 30`, RU:30), rollup older than 120 days (RU:31) and `auth_events` older than 90 days (RU:32). It runs at 01:30 IST (RU:17).
     - **(July item 7, purge half) fixed on main.** `c4ecdb6` (2026-09-22; ancestor of HEAD and origin/main) added `::int` to all three `make_interval` binds (RU:127, 131, 135) and to churn-risk (AS:378, 384).
     - Before that commit every nightly purge failed. Capture began `c33a556` on 2026-07-01, so prod raw `api_usage` held everything from 2026-07-01 until the first successful purge **after the image carrying `c4ecdb6` was deployed** (plus `API_USAGE_ROLLUP_ENABLED` ≠ `'false'`, RU:41).
     - Whether that happened is a prod fact. Q0 decides it: `min(created_at)` ≥ now−31d proves the fix is running *and* the queue is alive.
     - The portal's own CLAUDE.md still lists "churn-risk returns 400 'Database error' for every call" as an open upstream ask (v2.41.0, 2026-09-25, three days after the fix). The fix may not have been deployed at that time.
- **Internal accounts: no exclusion on any route** (AS:128–596). Bug class 4 applies to every number on the page.
- **Window semantics.** `window()` (AS:108–115) is a rolling `now − days×24h`, not IST day boundaries. The oldest IST bucket of every series is partial. This contradicts SUA §6 (see §D).

### A.1 TRUST — per-number table

Planes: A = domain tables, B = request capture, C = user/billing. "Excl." = exclusions applied. SQL ids refer to §A.3.

| # | Number as labelled on screen (UI file:line) | Plane | Our definition (route; file:line; SQL shape) | Their transformation (M; UI) | Window + TZ | Excl. | Draft verdict + why | SQL | Sev |
|---|---|---|---|---|---|---|---|---|---|
| 1 | **Studied · 24h / 7d / MAU · 30d** (E:473–475) | A | `summary`: `count(DISTINCT uid)` over the ENGAGED union (AS:99–106) with `ts >= now()-1d/7d/30d` (AS:135–140) | none; `nfmt` (E:39) | rolling 24h/7d/30d (labels say so) | none | **TRUE to its stated definition** (STUDIED_DEF E:28–32). But the definition undercounts by construction: (a) `user_question_attempts` is `@@unique([userId, questionId])` and is upserted (`schema.prisma:1111–1115`), so only the **first** answer to a question is a row; re-answers only bump `updated_at`; (b) `user_document_progress` is `@@unique([userId, documentId])` (`schema.prisma:1091`), so only the **first open** of a doc counts and continued reading is invisible; (c) PYQ practice is excluded even though it **persists** in `pyq_attempts` (`schema.prisma:1173–1198`, since `3c8bc41` 2026-08-26). Internal accounts are included. | E1, E1b | 🟠 |
| 2 | "opened the app: N" under Studied · 24h (E:473) | B | `summary.activeDau24h` = distinct `user_id` in raw `api_usage`, `created_at >= now()-1d` (AS:143–144) | none | rolling 24h, raw | none | **TRUE** ("any signed-in request", OPENED_DEF E:33–36). It excludes requests refused by a global guard (A.0 capture fact 1). Internal accounts are included. | E1 | 🟡 |
| 3 | **Opened-app MAU · 30d** (E:476) | B | distinct `user_id` in `api_usage_daily` with `day >= (now()-30d)::date` (AS:145–146) | none | ≈31 IST days (UTC-date cutoff vs IST `day`); today's row covers only 00:00–00:30 IST | none | **TRUE** (distinct users survive the rollup quirk). | E1 | 🟡 |
| 4 | **Stickiness** "studied 24h ÷ studied MAU" (E:477) | A | `engagedDau24h/engagedMau30d`, rounded to 3 dp; **0 when MAU = 0** (AS:147–149) | `×100` to one decimal (E:477) | as #1 | none | **TRUE.** A server-side 0-for-no-denominator (bug class 10) can only occur under an exam with no engaged users. | E1 | ⚪ |
| 5 | **DAU chart — "Studied" solid line** (E:161, 188) | A | `dau.engagedDau` = distinct uid per IST day `(ts+330m)::date`, `ts` in `[since, until]` (AS:177–181) | reversed to oldest→newest (E:123); **no zero-fill**: the server emits only days that have a row (AS:187–200), and the x-axis spaces points evenly (E:126) | `days` rolling; IST ✓; oldest bucket partial | none | **TRUE per point**, with the #1 undercount. A day with no engaged *and* no active rows vanishes and time compresses (only plausible under a small exam). | E2 | 🟡 |
| 6 | **DAU chart — "Opened the app" dashed line** (E:162, 190) | B | `api_usage_daily` distinct `user_id` per `day` (AS:182–186); `null` where no rollup row (AS:192) | null breaks the line (E:133) | `days` (rollup kept 120d, RU:31, so 60/90 are real); IST ✓ | none | **MISLABELLED at the right edge.** **Today's point = users active 00:00–00:30 IST only.** The rollup writes today at 00:30 IST (A.0 capture fact 3a), and after 05:30 IST `until::date` includes today. The dashed line therefore shows a near-zero cliff on the newest day, every day, unmarked. Earlier days are TRUE. A rollup outage longer than 2 days leaves permanent holes (2-day lookback). | E2 | 🟠 |
| 7 | **New signups / day** bars + "N new in Xd" (E:514–519) | C | `user_auth."createdAt"` bucketed IST, `[since, until]` (AS:326–330) | summed client-side (E:518); no zero-fill (AS:326–330, E:206) | rolling `days`; IST ✓ | none (internal and all statuses included) | **TRUE** (hard-deleted users drop out of history). | E3 | 🟡 |
| 8 | **Feature pull** bars: actions + "N users" per feature (E:530–534) | A | `features`: per-source rows `count(*)` and `count(DISTINCT uid)` (AS:221–232); keys `simulation, mcq, doc_reading, psychometric, ai_flashcard/ai_mnemonic/ai_revision` (`UserContentType` = FLASHCARD/MNEMONIC/REVISION, `schema.prisma:1368–1372`), `planner` | sorted by actions; labelled via `FEATURE_LABELS` (E:69–73) | `days` rolling | none | **MISLABELLED for two bars.** "Practice questions" counts **first answers only** (upsert), and "Doc reading" counts **docs opened for the first time** in the window. Both read as volume of use. PYQ practice is absent although persisted (hint E:527 only names chat/mains-eval/variation). | E3b | 🟠 |
| 9 | **Premium vs free · engagement (last 30 days)**: users, active, total actions, avg actions/active user per tier (E:544–578) | A + C | `premium-engagement`: no exam → tier = `premium_expires_at > now()` (AS:416); exam → live `user_exam_entitlements` row (AS:418–426); actions = ENGAGED since `window.since` (AS:429); avg = total/active, 0 if no active (AS:440) | relabelled "Premium (any purchase)" / "Entitled to X" (E:567–569); `toFixed(1)` | **follows `days`**, header says "last 30 days" (E:544, 546) | none | **MISLABELLED.** (a) The header and body copy say "last 30 days" but the window is the 7/30/60/90 selector (AS:412). (b) "Any purchase" includes MANUAL comps, sandbox grants and internal accounts. A `SUBSCRIBED` user with **NULL** `premium_expires_at` (perpetual; mirror written `longest.expiresAt ?? null`, `entitlement.service.ts:388`) lands in **Free**, unlike `derivePremiumState` (`premium-check.util.ts:313–318`). (c) Trials = free is stated correctly (E:556–557). A tier with zero users has no row, so the card silently disappears (E:562). | E4 | 🟠 |
| 10 | **Paywall hits · upgrade intent**: hits + users by route (E:585–600) | B | `paywall-hits`: raw `api_usage` with `status = 402 AND created_at >= since` (AS:456–460) | `humanRoute` is identity (E:74), so raw `/api/v1/...` templates; amber "(last 30d only — window not applied)" when days > 30 (E:587–594) | ≤30d raw (if the purge runs) | none | **WRONG (systematic omission).** Every 402 thrown by the **global `UsageQuotaGuard`** (UQG:150 PREMIUM_ONLY, UQG:322–334 QUOTA_EXHAUSTED) happens **before** the interceptor runs (GO:28–34), so it is never captured. That means `/api/v1/pyq/:id/reveal` (`pyq.controller.ts:436`), `/api/v1/mains/:id/reveal` (`mains.controller.ts:273`) and `/api/v1/content-doc/:id` (`content-doc.controller.ts:115`) — the main paywalls, and the doc's own example rows (FG §③) — can never appear. Only handler-thrown 402s show: `/quota/begin` (`quota.service.ts:182, 214`; all AI features collapse into one row because the route is grouped without `api_usage.feature`) and `/pyq/:id/details` (`free-question-set.service.ts:134`). The "last 30d" note is TRUE **only if** Q0 shows the purge running; otherwise 60/90 return up to ~125 days and the note is false. | E5, Q0 | 🔴 |
| 11 | **Retention curve**: cohort, D1/D7/D30 rings + % (E:610–635) | A + C | `retention`: cohort = `user_auth` created in `[since, until]`; dN = cohort users with any ENGAGED row at `ts >= signup + N days`, ever after (AS:347–357); pct = 0 if cohort = 0 (AS:358) | Ring value + `pct()` (E:620–624); explainer copy (E:630–635) | `days` rolling | none | **MISLABELLED.** The cumulative definition is disclosed correctly (E:631–632). But "read low on short windows" understates it: on the **default 30-day window D30 is identically 0** (every cohort member has `sd > now−30d`, so `sd+30d > now`), and on 7 days D7 is identically 0. The rings show "0 · 0%" as if measured. Re-reads (#1b) and PYQ-only users count as not retained. | E6 | 🟠 |
| 12 | **Churn risk · win-back list**: name, email, phone, "Nd ago" (E:644–686) | A | `churn-risk`: last ENGAGED `ts` per user, `< now − inactiveDays` (make_interval, `::int` ✓) and `>= now − 90d` (AS:374–385) | relTime of `lastActivity` (E:672), with no "last studied" label | inactiveDays 7/14/30 | none | **MISLABELLED as a win-back list.** "Silent" means "no new ENGAGED row". A user who opens the app daily, practises PYQs (persisted in `pyq_attempts`) or keeps reading an already-opened doc is listed as slipping. Internal accounts are listed. **Route now works on main** (`c4ecdb6`); deploy is unproven (Q0). | E7 | 🟠 |
| 13 | **Activity heatmap · IST** (E:698–718) | B | `heatmap`: raw `api_usage` **request count** per IST dow × hour, `created_at >= since` (AS:481–487) | 7×24 grid; dow 0 = Sun matches `days[]` (E:339); cell title "… IST — N" (E:354) | ≤30d raw; IST ✓ | none | **TRUE as request volume**, but the card never says "requests" (one user polling ≈ many cells). The clamp note has the same purge dependency as #10. Footer "Sparse early — it densifies…" (E:718) is stale: capture has run since 2026-07-01. | E8 | 🟡 |
| 14 | **Platforms**: iOS / Android / Web count + % (E:723–739) | C | `platforms`: `count(DISTINCT "userId")` per `lower(platform)` over `device_tokens WHERE "isActive"`; unknown platforms dropped (AS:531–536) | % of the **sum** (E:728) | **no window** (all-time live tokens) under a page-wide 7/30/60/90 selector | none | **MISLABELLED.** This is "users holding a still-valid push token", not platform usage. Tokens are only deactivated when a per-user FCM send fails (`notifications.service.ts:201–205`); `lastUsedAt` is set only at registration (:83–91). Multi-platform users count twice, so % is of registrations. Same source as `/app-users?platform=` (`sme-user.service.ts:119–121`, `isActive` + exact match), so the two agree except `lower()` vs exact case (DTO `IsIn` lowercase, `register-device-token.dto.ts:11–14`). | E9 | 🟡 |
| 15 | **Content & mocks → Top documents** (title + readers) (E:753–761) | A | `content.topDocuments`: `count(*)` of `user_document_progress` rows **created** in window, grouped by `cd.title, cd.subject`, top 10 (AS:562–566) | top 5; `subject` not rendered | `days` | none | **MISLABELLED.** Unique (user, doc) means "readers" = **new readers who first opened it in the window**. Two docs with the same title+subject merge. Under exam the filter is the content's tags (disclosed E:771–775). | E10 | 🟡 |
| 16 | **mock completion** gauge + submitted/started (E:765–766) | A | `simulation_attempts` started in window; submitted = `"submittedAt" IS NOT NULL`; rate = 0 if none started (AS:567–579) | `Math.round(rate||0)` | `days` | none | **TRUE.** 0% with 0/0 is ambiguous (bug class 10). Attempts started near the window end and submitted after it count as not submitted (documented, DEC §2). | E10 | ⚪ |
| 17 | **Push effectiveness**: read rate, sent, read (E:779–798) | C | `notification_history` rows in window; read = `"isRead"`; rate = 0 if none sent (AS:590–595) | caption "Read = opened the in-app feed afterwards, not tapped this push" (E:795) | `days` | none | **MISLABELLED (partly disclosed).** The caption is correct. But "sent" is history rows, so FCM **topic broadcasts** are absent (DEC §2), and the window includes unmatured rows (today's sends depress the rate). The UI points to Notification FX for the matured rate (E:796). | E11 | 🟡 |
| 18 | **Traffic by screen · top routes**: route, calls, users (E:803–818) | B | `endpoints`: `api_usage_daily` `sum(count)`, `count(DISTINCT user_id)` per route, `day ∈ [since::date, until::date]`, top 100 (AS:505–509) | top **20** rendered; raw template (E:812) | `days` (rollup 120d ✓) | none | **MISLABELLED + undercounted.** These are API route templates with the `/api/v1` prefix, not screens. **Calls** lose the 00:00–00:30 IST requests of every day except yesterday (A.0 capture fact 3b) and include today's 30-minute stub. Users: TRUE. Guard-refused requests are absent. | E12 | 🟡 |
| 19 | **User explorer → "Actions 30d"** column (E:907, 926) | A | `users.actions30d` = count of ENGAGED rows since `window(q).since` (AS:257, 289) | `nfmt` | **the 7/30/60/90 selector**, not 30 days | none | **MISLABELLED.** (July item 5 re-verified on current code.) It does **not** count authenticated API requests (the brief's premise). It counts plane-A rows, and the tooltip (E:907) describes that correctly. But the window follows `days` (`getUsageUsers({days,…})` E:395), so on 90d the column is "Actions 90d". It also has the #1 first-answer/first-open undercount. Sort "Most active" orders by this column with no tiebreak (AS:278), so offset pages can repeat or skip users on ties. | E13 | 🟠 |
| 20 | **User explorer → crown + "Premium until (any exam)"** (E:907, 915–919, 927) | C | `isPremium`: no exam → `premium_expires_at > now` (AS:302–304); **exam → live entitlement in THAT exam** (AS:265–273, 302–303). `premiumExpiresAt` = the mirror (AS:284) | crown tooltip "Premium on ANY exam (paid or trial)"; column shows **any** non-null date in amber, else "free" | now | none | **MISLABELLED.** Trials have no `premium_expires_at` (`derivePremiumState` treats it as `everPaid`, `premium-check.util.ts:308–328`), so "(paid or trial)" is false in both tooltips. Under `?exam=` the crown means "entitled to this exam", not "ANY exam". An **expired** date (a churned user) renders in premium-amber under "Premium until". | E13 | 🟠 |
| 21 | **User explorer → "Last seen"**, total / paging, search (E:891–941) | C | `max(sessions."lastAccessedAt")`, bumped on **every authenticated request** (`jwt.strategy.ts:57–61`); total = all `user_auth` matching search (+exam) (AS:293–295); search `ILIKE '%q%'` on email/name (AS:277, 290) | relTime; shared `Pagination` | full history | none | **TRUE.** `%`/`_` in search act as wildcards. Internal accounts are listed and counted in `total`. | E13 | ⚪ |

### A.2 What the `exam` selector does to each card

The selector is global and lives in the URL (`useExamParam`, XS:14–29). No exam = no param = byte-identical all-exams bodies (AC:35–36). An unknown slug gives the `UnknownExamState` page (E:430–437).

| Card | Exam axis (AS line) | Shape change | "since" chip / `unknownRows` rendered? | Retro-attribution note? | Finding |
|---|---|---|---|---|---|
| Pulse (summary) | **MIXED.** engaged = PERSON (AS:131–133); active = REQUEST (AS:134) | + top-level `examCoverage` (AS:153–158) | chip on the section header (E:469). `unknownRows` is in the tooltip, and also visible as "· N excluded" when `since` is non-null (XS:41–64) | ✓ (E:479–484) | Coverage is computed over **raw** `api_usage` 30d (AS:155–157), while `activeMau30d` reads the **rollup** (AS:145–146), so it does not describe the source it sits on. Harmless now: no NULL-exam rows remain in either 30-day window. ⚪ |
| DAU | MIXED (AS:173–176) | wrapped (AS:202–207) | chip outside `CardState` (E:502–507) ✓ | ✓ (E:510) | `since` = earliest exam-stamped **day inside the window**. For any window starting after ~2026-09-21 (every 7d/30d view today) that equals the **window start**, yet the chip says "Per-exam data from <date>" and the hint says "the dashed line only starts here" (E:505). That is false: it only starts there because the window does. 🟡 |
| New signups | PERSON (AS:323–325) | — | n/a | ✓ (E:520) | — |
| Feature pull | PERSON (AS:218–220) | — | n/a | ✓ (E:535) | — |
| Premium vs free | population PERSON; tier ENTITLEMENT (AS:418–426) | meaning changes | n/a | ✓ relabelled (E:551–554, 567–569) | Correct per FG §③. |
| Paywall hits | REQUEST (AS:455) | wrapped (AS:461–467) | ✓ (E:596) | n/a; "Filtered on the exam recorded on each refused request" (E:599) ✓ | `upsc-cse` includes every request without `X-Exam`/`?exam=`: the middleware defaults to it (`exam-context.middleware.ts`, catch → `DEFAULT_EXAM_ID`). |
| Retention | PERSON cohort (AS:343–345) | — | n/a | ✓ (E:636) | — |
| Churn risk | PERSON (AS:366–368) | — | n/a | ✓ (E:651) | — |
| Heatmap | REQUEST (AS:480) | wrapped (AS:488–492) | ✓ (E:709–714) | n/a | Same `since` = window-start issue as DAU; the "sparse grid" hint (E:712) is wrong today. 🟡 |
| Platforms | PERSON = token owner (AS:528–530) | — | n/a | ✓ (E:738) | Not windowed (#14). |
| Content & mocks | **CONTENT** tags, `*` = all (AS:551–560) | — | n/a | content note ✓ (E:771–775) | — |
| Push effectiveness | PERSON = recipient (AS:587–589) | — | n/a | ✓ (E:798) | — |
| Traffic by route | REQUEST via rollup (AS:504) | wrapped (AS:510–516) | ✓ (E:804) | none | No line saying the filter is the request's exam. ⚪ |
| User explorer | PERSON (AS:259–261); **`isPremium` re-scoped to entitlement** (AS:265–305) | meaning changes | n/a | ✓ (E:897) | The crown tooltip still says "ANY exam" (#20). 🟠 |
| Video · reels | **ignores exam** (E:387 passes only `days`) | — | — | **none** | Under an exam the video block silently shows all-exam Mux data with no "all exams" label. 🟡 |

- **The "since <date>" label is rendered** wherever `examCoverage` exists (5/5 routes on this page).
- **`unknownRows` is rendered**: always in the chip tooltip, and visibly as "· N excluded" when `since` is non-null and the count is > 0 (XS:61–63).
- `since: null` renders "Not recorded for this exam yet" (XS:47–56), and the empty states use `notRecorded()` (E:424–425). ✔ Contract met.

### A.3 Reconcile SQL (read-only, production Postgres; NOT run)

Prepend **Q0p** to every E-query.
- Each query reports two figures: **portal-equivalent** (what the card shows) and **real** (internal accounts and sandbox testers excluded).
- All DateTime columns are `timestamp without time zone` holding UTC. The service relies on the session TimeZone being UTC, so the preamble sets it explicitly (a session setting, not a write).
- For an exam-scoped run:
  - PERSON: add `AND NOT EXISTS (SELECT 1 FROM user_profiles ep WHERE ep."userId" = <uid> AND ep.active_exam_id IS NOT NULL AND ep.active_exam_id <> 'upsc-cse')` for upsc-cse, or `AND EXISTS (… ep.active_exam_id = '<slug>')` for any other exam (mirrors `sme-exam-scope.util.ts`).
  - REQUEST: add `AND exam_id = '<slug>'`.

```sql
-- Q0  Capture-plane health. Decides #10/#13 notes and whether c4ecdb6 is live. Run alone.
SET TIME ZONE 'UTC';
SELECT min(created_at) AS raw_min, max(created_at) AS raw_max, count(*) AS raw_rows,
       count(*) FILTER (WHERE exam_id IS NULL)          AS raw_null_exam,
       count(*) FILTER (WHERE created_at < now() - interval '31 days') AS raw_older_than_31d
FROM api_usage;
--   raw_min >= now()-31d  ⇒ the purge runs ⇒ the c4ecdb6 image is deployed and the BullMQ queue is alive.
--   raw_min ≈ 2026-07-01  ⇒ the purge never succeeded; paywall/heatmap 60/90 windows are REAL and the
--                           "(last 30d only — window not applied)" notes (E:587–594, 700–707) are FALSE.
SELECT min(day) AS daily_min, max(day) AS daily_max,
       min(day) FILTER (WHERE exam_id IS NOT NULL) AS first_exam_day,
       count(*) FILTER (WHERE exam_id IS NULL)     AS daily_null_exam
FROM api_usage_daily;                     -- daily_min >= current_date-121 ⇒ rollup purge works
SELECT min(created_at) FROM auth_events;  -- >= now()-91d ⇒ auth_events purge works
-- Rollup holes (days the 2-day lookback never healed), last 120 days:
SELECT d::date AS missing_day
FROM generate_series((SELECT min(day) FROM api_usage_daily), current_date - 1, interval '1 day') d
WHERE NOT EXISTS (SELECT 1 FROM api_usage_daily x WHERE x.day = d::date);

-- Q0p  Shared preamble for E1–E13
SET TIME ZONE 'UTC';
WITH internal AS (
  SELECT id FROM user_auth
  WHERE lower(email) LIKE '%@stanzasoft.com' OR lower(email) LIKE '%@stanzasoft.ai'
     OR lower(email) ~ '@prepmonkey\.'      OR lower(email) LIKE '%@designasylum.in'
  UNION SELECT DISTINCT "userId" FROM orders WHERE is_test        -- sandbox testers
),
engaged AS (                                   -- = AS:99–106 verbatim
  SELECT "userId" AS uid, "startedAt" AS ts FROM simulation_attempts
  UNION ALL SELECT user_id,  created_at  FROM user_content
  UNION ALL SELECT user_id,  created_at  FROM user_question_attempts
  UNION ALL SELECT user_id,  created_at  FROM custom_tasks WHERE source_id IS NULL
  UNION ALL SELECT "userId", "createdAt" FROM psychometric_test_results
  UNION ALL SELECT user_id,  created_at  FROM user_document_progress
)
-- ↓ append one of the following SELECTs

-- E1  Pulse (#1–#4)
SELECT
  count(DISTINCT uid) FILTER (WHERE ts >= now()-interval '1 day')                                          AS studied_24h,
  count(DISTINCT uid) FILTER (WHERE ts >= now()-interval '1 day'  AND uid NOT IN (SELECT id FROM internal)) AS studied_24h_real,
  count(DISTINCT uid) FILTER (WHERE ts >= now()-interval '7 days')                                         AS studied_7d,
  count(DISTINCT uid) FILTER (WHERE ts >= now()-interval '7 days' AND uid NOT IN (SELECT id FROM internal)) AS studied_7d_real,
  count(DISTINCT uid) FILTER (WHERE ts >= now()-interval '30 days')                                        AS studied_30d,
  count(DISTINCT uid) FILTER (WHERE ts >= now()-interval '30 days' AND uid NOT IN (SELECT id FROM internal)) AS studied_30d_real,
  (SELECT count(DISTINCT user_id) FROM api_usage WHERE created_at >= now()-interval '1 day')               AS opened_24h,
  (SELECT count(DISTINCT user_id) FROM api_usage WHERE created_at >= now()-interval '1 day'
      AND user_id NOT IN (SELECT id FROM internal))                                                        AS opened_24h_real,
  (SELECT count(DISTINCT user_id) FROM api_usage_daily WHERE day >= (now()-interval '30 days')::date)      AS opened_mau,
  (SELECT count(DISTINCT user_id) FROM api_usage_daily WHERE day >= (now()-interval '30 days')::date
      AND user_id NOT IN (SELECT id FROM internal))                                                        AS opened_mau_real
FROM engaged;

-- E1b  What "Studied" would be with PYQ practice + continued reading/re-answering (sizes the #1/#8/#19 undercount)
SELECT count(DISTINCT uid) FILTER (WHERE ts >= now()-interval '30 days') AS studied_30d_broad
FROM (SELECT uid, ts FROM engaged
      UNION ALL SELECT user_id, created_at FROM pyq_attempts
      UNION ALL SELECT user_id, updated_at FROM user_document_progress
      UNION ALL SELECT user_id, updated_at FROM user_question_attempts) b
WHERE uid NOT IN (SELECT id FROM internal);

-- E2  DAU (#5, #6): rollup vs raw per IST day, last 7 days → today's stub + the 00:00–00:30 loss
SELECT r.d AS ist_day, r.raw_users, r.raw_calls, r.calls_0000_0030,
       x.rollup_users, x.rollup_calls
FROM (SELECT (created_at + interval '330 minutes')::date AS d,
             count(DISTINCT user_id) AS raw_users, count(*) AS raw_calls,
             count(*) FILTER (WHERE (created_at + interval '330 minutes')::time < time '00:30') AS calls_0000_0030
      FROM api_usage WHERE user_id IS NOT NULL AND created_at >= now() - interval '8 days' GROUP BY 1) r
LEFT JOIN (SELECT day AS d, count(DISTINCT user_id) AS rollup_users, sum(count) AS rollup_calls
           FROM api_usage_daily WHERE day >= current_date - 8 GROUP BY 1) x USING (d)
ORDER BY 1;
-- expect: rollup_users == raw_users except TODAY (tiny); rollup_calls ≈ raw_calls − calls_0000_0030
-- for days older than yesterday.
-- engaged series, IST, 30d, portal-equivalent and real:
SELECT (ts + interval '330 minutes')::date AS ist_day, count(DISTINCT uid) AS studied,
       count(DISTINCT uid) FILTER (WHERE uid NOT IN (SELECT id FROM internal)) AS studied_real
FROM engaged WHERE ts >= now() - interval '30 days' GROUP BY 1 ORDER BY 1 DESC;

-- E3  Signups (#7)
SELECT ("createdAt" + interval '330 minutes')::date AS ist_day, count(*) AS new_users,
       count(*) FILTER (WHERE id NOT IN (SELECT id FROM internal)) AS new_users_real
FROM user_auth WHERE "createdAt" >= now() - interval '30 days' GROUP BY 1 ORDER BY 1 DESC;

-- E3b  Feature pull (#8): first-touch rows (what the card counts) vs rows touched in the window
SELECT 'mcq'         AS f, count(*) FILTER (WHERE created_at >= now()-interval '30 days') AS counted,
                            count(*) FILTER (WHERE updated_at >= now()-interval '30 days') AS touched
FROM user_question_attempts WHERE user_id NOT IN (SELECT id FROM internal)
UNION ALL
SELECT 'doc_reading', count(*) FILTER (WHERE created_at >= now()-interval '30 days'),
                      count(*) FILTER (WHERE updated_at >= now()-interval '30 days')
FROM user_document_progress WHERE user_id NOT IN (SELECT id FROM internal)
UNION ALL
SELECT 'pyq_attempts (not on card)', count(*), count(DISTINCT user_id)
FROM pyq_attempts WHERE created_at >= now()-interval '30 days' AND user_id NOT IN (SELECT id FROM internal);

-- E4  Premium vs free (#9)
SELECT CASE WHEN u.premium_expires_at > now() THEN 'premium' ELSE 'free' END AS tier,
       (u.id IN (SELECT id FROM internal)) AS is_internal,
       count(*) AS users, count(*) FILTER (WHERE e.actions > 0) AS active_users,
       COALESCE(sum(e.actions), 0) AS total_actions
FROM user_auth u
LEFT JOIN (SELECT uid, count(*) AS actions FROM engaged WHERE ts >= now()-interval '30 days' GROUP BY uid) e ON e.uid = u.id
GROUP BY 1, 2 ORDER BY 1, 2;
-- "premium (any purchase)" with no real-money paid order:
SELECT count(*) FROM user_auth u
WHERE u.premium_expires_at > now()
  AND NOT EXISTS (SELECT 1 FROM orders o WHERE o."userId" = u.id AND o.status = 'PAID'
                    AND NOT o.is_test AND o.payment_source <> 'MANUAL');
-- perpetual premium shown as FREE:
SELECT count(*) FROM user_auth WHERE status = 'SUBSCRIBED' AND premium_expires_at IS NULL;

-- E5  Paywall hits (#10) — prove the guard-thrown 402s are missing
SELECT route, feature, count(*) AS hits, count(DISTINCT user_id) AS users
FROM api_usage WHERE status = 402 AND created_at >= now() - interval '30 days'
GROUP BY 1, 2 ORDER BY 3 DESC;
-- expect ONLY /api/v1/quota/begin and /api/v1/pyq/:id/details; NO /pyq/:id/reveal, /mains/:id/reveal, /content-doc/:id
SELECT route, status, count(*) FROM api_usage
WHERE route IN ('/api/v1/pyq/:id/reveal', '/api/v1/mains/:id/reveal', '/api/v1/content-doc/:id')
  AND created_at >= now() - interval '30 days' GROUP BY 1, 2 ORDER BY 1, 2;   -- expect no 402 rows at all
-- independent signal for the same window (client-emitted, a floor):
SELECT count(*) AS paywall_viewed, count(DISTINCT user_id) AS users
FROM auth_events WHERE event_type = 'paywall_viewed' AND created_at >= now() - interval '30 days'
  AND (user_id IS NULL OR user_id NOT IN (SELECT id FROM internal));

-- E6  Retention (#11) — prove D30 is structurally 0 on the default window, then the age-matched figure
SELECT count(*) AS cohort_30d,
       count(*) FILTER (WHERE "createdAt" + interval '30 days' <= now()) AS eligible_for_d30   -- expect 0
FROM user_auth WHERE "createdAt" >= now() - interval '30 days';
WITH cohort AS (SELECT id, "createdAt" AS sd FROM user_auth
                WHERE "createdAt" >= now()-interval '60 days' AND "createdAt" < now()-interval '30 days'
                  AND id NOT IN (SELECT id FROM internal))
SELECT count(*) AS cohort_real,
       count(*) FILTER (WHERE EXISTS (SELECT 1 FROM engaged a WHERE a.uid = c.id AND a.ts >= c.sd + interval '1 day'))   AS d1,
       count(*) FILTER (WHERE EXISTS (SELECT 1 FROM engaged a WHERE a.uid = c.id AND a.ts >= c.sd + interval '7 days'))  AS d7,
       count(*) FILTER (WHERE EXISTS (SELECT 1 FROM engaged a WHERE a.uid = c.id AND a.ts >= c.sd + interval '30 days')) AS d30
FROM cohort c;

-- E7  Churn risk (#12): how many "slipping" users are actually active
WITH la AS (SELECT uid, max(ts) AS last_ts FROM engaged GROUP BY uid),
     churn AS (SELECT uid FROM la WHERE last_ts < now() - interval '14 days' AND last_ts >= now() - interval '90 days')
SELECT count(*) AS listed,
       count(*) FILTER (WHERE uid IN (SELECT id FROM internal)) AS internal_listed,
       count(*) FILTER (WHERE EXISTS (SELECT 1 FROM api_usage a WHERE a.user_id = churn.uid
                                       AND a.created_at >= now() - interval '14 days')) AS opened_app_since,
       count(*) FILTER (WHERE EXISTS (SELECT 1 FROM pyq_attempts p WHERE p.user_id = churn.uid
                                       AND p.created_at >= now() - interval '14 days')) AS practised_pyq_since
FROM churn;

-- E8  Heatmap (#13), real
SELECT extract(dow FROM created_at + interval '330 minutes')::int AS weekday,
       extract(hour FROM created_at + interval '330 minutes')::int AS hour,
       count(*) AS requests, count(DISTINCT user_id) AS users
FROM api_usage WHERE created_at >= now() - interval '30 days' AND user_id NOT IN (SELECT id FROM internal)
GROUP BY 1, 2 ORDER BY 1, 2;

-- E9  Platforms (#14)
SELECT lower(platform) AS platform, count(DISTINCT "userId") AS users_card,
       count(DISTINCT "userId") FILTER (WHERE "userId" NOT IN (SELECT id FROM internal)) AS users_real,
       count(DISTINCT "userId") FILTER (WHERE EXISTS (SELECT 1 FROM api_usage a WHERE a.user_id = t."userId"
                                         AND a.created_at >= now() - interval '30 days')) AS seen_30d
FROM device_tokens t WHERE "isActive" GROUP BY 1;
SELECT count(*) AS multi_platform_users FROM (
  SELECT "userId" FROM device_tokens WHERE "isActive" GROUP BY 1 HAVING count(DISTINCT lower(platform)) > 1) x;

-- E10  Content & mocks (#15, #16)
SELECT cd.id, cd.title, cd.subject,
       count(*) FILTER (WHERE p.created_at >= now()-interval '30 days') AS new_readers_card,
       count(*) FILTER (WHERE p.updated_at >= now()-interval '30 days') AS readers_touched
FROM user_document_progress p JOIN content_documents cd ON cd.id = p.document_id
WHERE p.updated_at >= now()-interval '30 days' AND p.user_id NOT IN (SELECT id FROM internal)
GROUP BY 1, 2, 3 ORDER BY 4 DESC LIMIT 10;
SELECT title, subject, count(*) FROM content_documents GROUP BY 1, 2 HAVING count(*) > 1;   -- merge risk
SELECT count(*) AS started, count(*) FILTER (WHERE "submittedAt" IS NOT NULL) AS submitted
FROM simulation_attempts WHERE "startedAt" >= now()-interval '30 days' AND "userId" NOT IN (SELECT id FROM internal);

-- E11  Push effectiveness (#17): raw vs matured (≥3 days old)
SELECT count(*) AS sent, count(*) FILTER (WHERE "isRead") AS read,
       count(*) FILTER (WHERE "createdAt" < now()-interval '3 days') AS sent_matured,
       count(*) FILTER (WHERE "isRead" AND "createdAt" < now()-interval '3 days') AS read_matured,
       count(*) FILTER (WHERE "userId" NOT IN (SELECT id FROM internal)) AS sent_real
FROM notification_history WHERE "createdAt" >= now() - interval '30 days';

-- E12  Endpoints (#18): rollup calls vs raw calls per route, last 7 complete IST days
SELECT r.route, r.raw_calls, x.rollup_calls, r.raw_calls - COALESCE(x.rollup_calls, 0) AS lost
FROM (SELECT route, count(*) AS raw_calls FROM api_usage
      WHERE user_id IS NOT NULL
        AND (created_at + interval '330 minutes')::date BETWEEN current_date - 8 AND current_date - 2
      GROUP BY 1) r
LEFT JOIN (SELECT route, sum(count) AS rollup_calls FROM api_usage_daily
           WHERE day BETWEEN current_date - 8 AND current_date - 2 GROUP BY 1) x USING (route)
ORDER BY r.raw_calls DESC LIMIT 20;

-- E13  User explorer (#19–#21): one page as the card sees it (days=30, sort=actions) + mislabel counts
SELECT u.id, u.email, u.premium_expires_at,
       (u.premium_expires_at IS NOT NULL AND u.premium_expires_at <= now()) AS expired_shown_amber,
       s.last_seen, COALESCE(a.n, 0) AS actions_window
FROM user_auth u
LEFT JOIN (SELECT "userId" AS uid, max("lastAccessedAt") AS last_seen FROM sessions GROUP BY 1) s ON s.uid = u.id
LEFT JOIN (SELECT uid, count(*) AS n FROM engaged WHERE ts >= now()-interval '30 days' GROUP BY 1) a ON a.uid = u.id
ORDER BY actions_window DESC, u.id LIMIT 20;
SELECT count(*) FILTER (WHERE premium_expires_at <= now()) AS expired_dates_rendered_amber,
       count(*) FILTER (WHERE status = 'ACTIVE' AND premium_expires_at IS NULL
                          AND now() < COALESCE(trial_ends_at, "createdAt" + interval '14 days')) AS trial_users_shown_free
FROM user_auth;
```

## B. MEANING

> Owner ruling 2026-11-03 (verbatim): "The numbers are there BUT those numbers are not explaining anything — same on the aspirant profile screen. No need to drastically change stuff; compact the details or combine and show. I don't need to see the endpoints a user is hitting on the portal, it explains nothing. Instead explain in simple terms: some X metric done on this data, or 'he stayed X mins', or 'he has been using this many times'. I may be grossly oversimplifying — the portal might already do some things well and corrections may not be needed at all. Just analyze and judge."

### B.1 Per-card plain-language test and bucket

**Verdict in one line.** About half the page already passes the test and should stay. Three things fail it:
- **Traffic by route** — raw noise.
- **Activity heatmap** — request counts with no unit.
- **Pulse + Platforms** — tiles a reader cannot turn into a sentence without a merge or relabel.

Every number inside the quoted sentences below is an **illustrative placeholder, not a production value**. No SQL was run.

The derived metrics the owner asks for are mostly computable today. The pattern already exists in the portal: `session-utils.ts:26` `SESSION_GAP_MINUTES = 30` builds per-user sessions client-side on the profile.

| # | Card (E: line) | The one sentence a non-engineer can say today | Bucket | What to do (minimal) |
|---|---|---|---|---|
| 1 | **Pulse**: Studied 24h / 7d / MAU, Opened-app MAU, Stickiness (E:472–477) | Barely. "X people studied in the last 30 days and Y opened the app" needs four tiles and two hover tooltips to assemble. | **COMPACT/COMBINE** | Merge into one sentence tile: "**N studied this month (M opened the app); on a typical day D of them study (S %)**". Keep the numbers; drop the tile-per-number layout. Add the derived "**median days studied per active user (30d)**" (B.1a M1). The data is already on the page except M1. |
| 2 | **Daily active users** chart, studied vs opened (E:497–512) | "About 300 study and 900 open the app on a typical day; the gap is people who open and leave." Passes, except the misleading right-edge point (A #6). | **KEEP AS IS** | Keep. Only fix: draw today as partial or drop it (A #6). No redesign. |
| 3 | **New signups / day** (E:513–522) | "We got 1,240 new signups in the last 30 days, peak 96 on 12 Oct." Passes. | **KEEP AS IS** | — |
| 4 | **Feature pull** (E:527–538) | "Planner and practice questions are the most-used features: 480 and 120 people." Passes in form, but the numbers are first-touch counts (A #8), and the sentence the owner wants ("how many times / do they come back") is missing. | **KEEP + one derived column** | Keep the bars. Add "**repeat rate**" per feature = share of users of feature X in the window who used it on ≥2 distinct IST days (B.1a M4). Optionally add PYQ practice as a bar (M5). |
| 5 | **Premium vs free** (E:543–581) | "A premium user does 40 study actions a month, a free one 14." Passes; the punchline (avg actions per active user) is the right shape. | **KEEP AS IS** (fix the "last 30 days" label, A #9) | Optional: swap "avg actions" for "**avg days studied**" (M1 split by tier), which is easier to say aloud. |
| 6 | **Paywall hits · upgrade intent** (E:582–601) | Intended: "180 free users hit the PYQ-reveal paywall this month." Fails today: it shows raw `/api/v1/...` route templates, and the biggest paywalls are missing (A #10). | **RAW/NOISE** (as rendered) | Replace route rows with "**N free users hit a paywall this month; top walls: PYQ reveal (a), AI flashcard (b) …**" = distinct users per *feature* (M6). Needs the guard-402 capture fix first (A #10); without it the sentence is false. |
| 7 | **Retention curve** D1/D7/D30 (E:608–641) | Intended: "Of people who joined 1–2 months ago, 40 % came back after a week." Fails on the default 30-day window: D30 is always 0 % (A #11), and "cohort 1,240 · D1 49" is not a sentence. | **COMPACT/COMBINE** | Same data. Compute each ring only over cohort members **old enough** to qualify (eligible denominator: `createdAt <= now − N days`), and render one sentence per ring: "**X % of people who joined ≥7 days ago studied again after day 7**". No new data. |
| 8 | **Churn risk · win-back list** (E:642–690) | "These 134 people studied regularly but have not studied for 2+ weeks — call them." Passes in form. But many listed people did open the app or practise PYQs (A #12), so the sentence is wrong for them. | **KEEP + relabel/derive** | Keep the list. Change each row's sub-line from "20d ago" to "**last studied 20d ago · last opened app 2d ago**" (M8). Optionally hide people who opened the app inside the inactivity window. Both are derivable today. |
| 9 | **Activity heatmap · IST** (E:697–720) | "Most activity is 8–11 pm on weekdays." Half-passes: the shape is meaningful, but a cell value is a request count, so the number itself explains nothing. | **COMPACT/COMBINE** (relabel) | Keep the grid. Change the cell value from request count to "**people active in this hour**" (distinct users, M7). Add one caption sentence: "Peak: Tue 21:00 IST, 140 people on an average week". Same table. |
| 10 | **Platforms** (E:722–743) | Fails as labelled: "1,280 Android" reads as active Android users, but it is users still holding a push token, all-time (A #14). | **COMPACT/COMBINE** (re-source) | Same card, re-sourced: "**Of people active this month: 70 % Android, 28 % iOS, 2 % web**" from `api_usage.platform` (M9). |
| 11 | **Content & mocks** (E:744–777) | "Most-opened doc this month: Fundamental Rights (88 new readers); 69 % of started mocks were finished." Passes once "readers" says "new readers" (A #15). | **KEEP AS IS** (relabel "new readers") | — |
| 12 | **Push effectiveness** (E:778–800) | "We sent 5,400 notifications and 24 % were followed by the person opening their notification feed." Passes, and the caption already says exactly that (E:795). The link to Notification FX for the matured rate is the right minimal move. | **KEEP AS IS** | — |
| 13 | **Traffic by screen · top routes** (E:802–821) | None. `/api/v1/tasks/today — 18,400 calls — 920 users` explains nothing to a shareholder. This is exactly what the owner called out. | **RAW/NOISE** | Remove from the main flow, or collapse behind a "for engineers" toggle. What it was trying to say is already said better by Feature pull (#4) plus the derived sentence "**the average active user had N sessions and ~M minutes in the app this month**" (M2, M3). |
| 14 | **User explorer** table (E:882–948) | Partly. "Meghana, last seen 2h ago, 84 actions" — "actions" is opaque and its window is mislabelled (A #19, #20). | **COMPACT/COMBINE** | Keep the table, but swap the "Actions 30d" column for "**studied on D of last 30 days · S sessions · ~M min**" (M1, M2, M3 per user), and keep Last seen (TRUE, A #21). The per-user session/minutes logic already exists in the portal (`session-utils.ts`), so this mirrors it in one SQL aggregate. |
| — | Video · reels (E:825–880) | Out of scope (Mux). Its tiles already read as sentences ("Total views · Unique viewers · Avg watch / view"). | KEEP (not audited) | — |

#### B.1a Derived metrics proposed above — computable today?

Retention caps:
- raw `api_usage`: 30 days (RU:30; if the purge runs, Q0);
- `api_usage_daily`: 120 days (RU:31), but only day × user × route × exam × count, with no time of day, platform or feature;
- `auth_events`: 90 days (RU:32);
- plane A: full history.

| Id | Metric | Computable today? | Source (table.column) and derivation rule | Cap / caveat |
|---|---|---|---|---|
| M1 | **Days studied** per user (and median per active user, split by tier) | **Yes** | `count(DISTINCT (ts + interval '330 minutes')::date)` over the ENGAGED union (AS:99–106); better with `pyq_attempts.created_at`, `user_question_attempts.updated_at`, `user_document_progress.updated_at` added | Full history. `updated_at` only keeps the *latest* touch per row, so it still undercounts repeat days on the same doc/question. |
| M1b | **Days active (opened app)** per user | **Yes** | `count(DISTINCT day)` from `api_usage_daily` per `user_id` | 120 days |
| M2 | **Sessions** per user | **Yes**, within 30 days | Order `api_usage` rows by `(user_id, created_at)` (optionally per `device_id`); a new session starts when the gap to the previous row is ≥ 30 min (the same `SESSION_GAP_MINUTES = 30` the profile uses, `session-utils.ts:26`). Can be sharpened with `auth_events` `app_opened` / `app_backgrounded` pairs (`event_type`, `user_id`, `created_at`). | 30-day raw cap. `app_opened`/`app_backgrounded` exist only from app 2.0 (2026-09-17) onward and are capped at 90 days. Requests refused by a guard are invisible (A.0 capture fact 1). |
| M3 | **Minutes active** per user ("he stayed X mins") | **Estimable today**; exact needs new persistence | Sum over M2 sessions of `last_row − first_row` (+ a fixed tail, as the profile estimate does); or the sum of `app_opened` → `app_backgrounded` spans where both exist. | Lower-bound estimate. A screen read without requests is invisible. **Exact foreground time needs new persistence**: the portal already lists "sessions endpoint + `foregroundMs`" as an upstream ask (portal CLAUDE.md v2.41). Flag only. |
| M4 | **Repeat rate** per feature | **Yes** | Per feature in the ENGAGED/features union (AS:221–232): users with `count(DISTINCT IST day) >= 2` ÷ users with ≥1 | Full history; the same first-touch caveat as M1 for `mcq` and `doc_reading` |
| M5 | **PYQ questions attempted / PYQ practisers** | **Yes** | `pyq_attempts` (`user_id`, `question_id`, `attempt_number`, `created_at`): `count(*)`, `count(DISTINCT user_id)`, re-attempts = `attempt_number > 1` | From 2026-08-26 (`3c8bc41`, backfilled from Neo4j) |
| M6 | **Distinct users who hit a paywall, by feature** | **Partly** | Handler 402s: `api_usage` rows with `status = 402`, grouped by `feature` (`/quota/begin`) or route (`/pyq/:id/details`), `count(DISTINCT user_id)`. Alternative floor: `auth_events.event_type='paywall_viewed'` (client-emitted, 2.0+, 90 days). | **Guard-thrown 402s (PYQ/Mains reveal, content-doc) are not persisted anywhere today: needs capture (flag only, A #10).** 30-day raw cap. |
| M7 | **People active per weekday × hour** | **Yes** | `count(DISTINCT user_id)` per IST dow × hour from raw `api_usage` (one-line change to AS:481–487) | 30 days |
| M8 | **Last studied vs last opened app** per user | **Yes** | `max(ts)` over ENGAGED (+`pyq_attempts`) vs `max(sessions."lastAccessedAt")` (updated on every authenticated request, `jwt.strategy.ts:57–61`) | Full history (sessions rows persist) |
| M9 | **Platform of active users** | **Yes** | `api_usage.platform` (from `x-platform`; INT:106), `count(DISTINCT user_id)` per platform in the window; NULL = pre-header client | 30 days; NULL share is reported separately, never folded |
| M10 | **Time to come back** ("returned after N days") | **Yes** | Per cohort user: `min(ts) − createdAt` for the first ENGAGED row at ≥1 day after signup | Full history |

### B.2 The six shareholder questions

| Q | Answered here? | What is missing | Data already persists? |
|---|---|---|---|
| (1) Are people using feature X, with repeat rate | **Partially.** Users + actions per feature (#8); no repeat rate. Actions ÷ users is intensity, not "came back". | Repeat rate (distinct IST days per user per feature, or week-over-week return). PYQ practice is missing entirely. Re-answers and continued reading are invisible (first-touch rows only). Chat / mains-eval / pyq-variation are not shown. | **Yes, for all but the Dify outputs.** `pyq_attempts.user_id, created_at, attempt_number`; `user_question_attempts.updated_at`; `user_document_progress.updated_at`. A union change, no new persistence. Dify features leave a **30-day** trace in raw `api_usage` (`route='/api/v1/quota/begin'`, `feature ∈ TOKEN_BEGIN_FEATURES`, INT:165–178), plus `auth_events.chat_conversation_started`. The rollup drops `feature`, so there is no history beyond 30 days. Chat content, mains-eval results and variations: no persistence (flag). |
| (2) Per-exam and per-language funnels | **Partially (exam only).** The global exam selector scopes every card (A.2). No funnel on this page. | Language dimension: none. Funnel lives on `/insights/purchase-funnel` and `/insights/onboarding-funnel`. | Exam: `user_profiles.active_exam_id`, `api_usage(.daily).exam_id`. Language: `user_profiles.ui_language` (`schema.prisma:213`) and content `language` columns exist, but no analytics route reads them (flag only). |
| (3) Where buyers stop in checkout | **Not answered.** Paywall hits (#10) is the only money-intent card, and it misses the guard-thrown 402s. | The checkout funnel itself; only linked from here under an exam (E:599). | `auth_events` (paywall_viewed … purchase_failed), `orders`. Not this page's job. |
| (4) Question quality | **Not answered.** | — | `/insights/question-quality` (other screen). |
| (5) Notification outcomes | **Partially.** Sent/read (#17), where read = bulk feed open, topic broadcasts are absent, and unmatured rows are included. | Per-push tap/open, per-rule outcomes. | `notification_history` (`isRead`, `read_at`, `rule_id`, `category`); per-rule `tapped` is on Notification Rules (other screen). |
| (6) Support view of one user's attempts incl. Razorpay mandates | **Partially.** User explorer + churn list deep-link to `/app-users/:id` (E:660–673, 913). Here: last seen, window actions, mirror expiry only. | Attempts, mandates and orders are not on this page. | Attempt tables and `orders`/`payments` all persist; the profile page is the place (other screen). |

## C. GAP

| Item | Source of the expectation | Status |
|---|---|---|
| `examCoverage.since` / `unknownRows` on summary, dau, paywall-hits, heatmap, endpoints | FG "Three things the client MUST handle" §2; SUA §0b | **rendered** (A.2). The chip's "from <date>" now usually equals the window start (A.2 DAU row). 🟡 |
| Wrapped `{data, examCoverage}` normalisation | FG §1; AC:40–43 | **rendered** (E:409–413, INS:70–77) |
| `sort` (lastSeen / actions) | SUA §1; DTO:69–72 | **rendered** (E:886–888) |
| `search` (email / name) | SUA §1 | **rendered** (E:891–892). Wildcards are not escaped (AS:277). ⚪ |
| `inactiveDays` | SUA §2; DTO:82–88 (1–90) | **rendered** as 7/14/30 (E:646) |
| `startDate` / `endDate` | SUA header, FG conventions | **proxied-only** (M:6216–6217, 6236). The UI never sends them. Our `window()` reads them as `new Date(str)` = UTC midnight, contradicting the "IST always" claim (§D). |
| `content.topDocuments[].subject` | FG coverage contract | **absent.** Fetched (API type), not rendered (E:755–760). The contract says every field must be surfaced. |
| `endpoints` rows 21–100 | FG coverage contract | **absent** (E:810 `slice(0, 20)`) |
| "Humanize … route templates in copy" | FG Hard reminders | **absent**: `humanRoute = (r) => r` (E:74); raw `/api/v1/...` templates are shown |
| `purchase-funnel` panel beneath Money | FG IA step 4 | **absent** without an exam; a link only under `?exam=` (E:599) |
| `api_usage.feature` for `/quota/begin` | INT:159–178, schema `ApiUsage.feature` | **absent on our side.** `paywall-hits` groups by `route` only (AS:457–460), so every AI-feature 402 collapses into one row. |
| Guard-refused requests (402 quota, 403 feature-disabled) | SUA §3/§0 "HTTP 402 hits" | **absent on our side** (A.0 capture fact 1). Affects paywall-hits here and `paywall402` in release-health (`sme-release-health.service.ts:148`). |
| Stale/false UI copy | — | "(paid or trial)" on crown and column tooltips (E:907, 916) — false. "ANY exam" crown under `?exam=` — false. "last 30 days" on the premium card and the "Actions 30d" header follow the selector instead (E:544, 546, 907). "Traffic by screen" (E:803) — these are routes. "the dashed line only starts here" (E:505) and "a sparse grid here means few days of per-exam data" (E:712) — false once the window is post-2026-09-21. "Sparse early — it densifies as more traffic is recorded" (E:718) and "request capture is recent" (E:716) — stale since July. "D7 and D30 read low on short windows" (E:631–635) — they are exactly 0. "(last 30d only — window not applied)" (E:592, 705) — true only if Q0 shows the purge running. |
| Portal docs vs us | portal `CLAUDE.md` v2.41 upstream asks | "churn-risk returns 400 'Database error' for every call" is **stale on main** (`c4ecdb6`). Live state depends on deploy (Q0). |
| Video section exam scope | FG "one global exam selector … scopes the whole console" | **absent**: no "all exams" note when an exam is picked (E:826–880). Mux side is out of scope. |

## D. Contradictions (code vs docs or brief)

1. **Brief / July item (5):** "Actions 30d counts authenticated API requests."
   - Code: it counts ENGAGED plane-A rows (AS:289), not `api_usage`.
   - It also does not cover 30 days: it uses `window(q).since` (AS:257), and the UI passes the page's `days` (E:395).
   - DEC §2 ("in the last 30 days") is also wrong about the window.
2. **Brief / July item (7) and DEC §8 "Still open #3":** "paywall-hits offered 7/30/60/90 with no note."
   - The note exists: E:587–594, added in `2f1f5fa` (2026-07-29), and a shorter "(last 30d only)" existed before that commit.
   - The open item is stale. It still depends on the purge actually running (Q0).
3. **SUA §3, DEC §2, FG §③: "paywall hits = HTTP 402 hits."**
   - Guard-thrown 402s are never captured (GO:28–34, UQG:150, 322–334, INT:120–125).
   - FG's example rows (`/pyq/:id/reveal`, `/mains/:id/reveal`) are exactly the ones that can never appear.
   - The same blind spot applies to SHARED_RULES' "plane B = authenticated requests": it is *authenticated requests that pass every global guard*.
4. **SUA §1 (users):** "`isPremium` … is **not** re-scoped to the exam."
   - Code re-scopes it to a live entitlement under `?exam=` (AS:265–273, 302–303; `f6329e6`, 2026-09-21), and AC:88 says so.
   - The doc was not updated in the same release.
5. **SUA §6 and FG conventions: "IST always (`days` boundaries resolve in IST)"; "startDate=YYYY-MM-DD".**
   - `window()` is rolling `now − days×86 400 000 ms` (AS:108–115), and `startDate` is parsed as UTC midnight.
   - Only the *buckets* are IST.
6. **DEC §2 retention: "a 30-day window always shows a *depressed* D7/D30."** On a ≤30-day window D30 is **exactly 0**, and on a 7-day window so is D7 (AS:352–356 vs AS:108–115).
7. **SUA §5 / DEC §0: "rollup collapses raw rows into `api_usage_daily`".**
   - It does, but `DO UPDATE SET count = EXCLUDED.count` with a 2-day lookback starting at 00:30 IST (RU:95–117) overwrites every day's count without its 00:00–00:30 IST requests, and leaves today as a 30-minute stub.
   - Nothing in the docs or UI says so.
8. **`schema.prisma:721` comment** ("route = `/pyq/:id/reveal`") and FG examples (`/tasks/today`, `/pyq/:id/reveal`) conflict with stored values, which carry the `/api/v1` prefix (INT:137–141).
9. **Brief:** "the `{data, examCoverage}` wrapping at L40–43" — those lines are the controller's doc comment. The wrapping is implemented in the service at AS:201–207, 461–467, 488–492, 510–516.
10. **Brief:** "the nightly purge's same bug — so `api_usage` may hold MORE than 30 days on prod."
    - The purge bug **was** fixed in the same commit `c4ecdb6` (RU:127, 131, 135).
    - `api_usage` holds more than 30 days only if that image was deployed late or not at all, or the queue is disabled. Q0 settles it.
    - Until then: since 2026-07-01 at most, and in steady state ≤31 days.
11. **UI copy vs UI copy on the same page:** the premium panel says active trials count as *free* (E:556–557), while the user-explorer tooltips say premium is "paid **or trial**" (E:907, 916). The code (AS:302–304) agrees with the panel.
12. **Portal `CLAUDE.md` v2.30.1:** "/engagement is admin + app_admin + analytics". `require_analytics_access` also admits `app_admin_users` and `sales` (`app/auth.py:419–423`). ⚪

## E. Processes stopped and files written

- Processes started: **none** (pids: none). No SQL run, no network, no git fetch/pull/checkout.
- Files written: `/private/tmp/claude-502/-Users-yashwanth-Desktop-projects/b0942abe-5468-460e-92b6-db352e6a3606/scratchpad/phase1/S02_engagement.md` (this report).
