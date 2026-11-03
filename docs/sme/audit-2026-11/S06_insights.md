# S06 — the five `/insights/*` pages (Phase 1 verifier)

Read-only pass, 2026-11-03.
- Portal at `7b00daf` (v2.44.0). Backend checkout at `cca9ede` (docs-only on top of the v2.44 code). Live evidence comes from the 14:20–14:21 IST captures in `.playwright-mcp/live/insights_*`.
- **Prod SQL was run read-only.** The lane brief explicitly allows it, which overrides SHARED_RULES "No prod/staging SQL in this pass". All of it was SELECT through `scripts/staging-db.ts --prod`. Every reconcile query is pinned to the capture instant (2026-11-03 08:50–08:51 UTC), so most numbers match the screen exactly.

Path keys:
- `QQ:` `RH:` `NFX:` `ONB:` `PF:` = `upsc-document-generator/admin-ui/src/pages/Insight{QuestionQuality,ReleaseHealth,NotificationFx,OnboardingFunnel,PurchaseFunnel}.tsx`
- `INS:` = `admin-ui/src/services/insights.api.ts`; `API:` = `admin-ui/src/services/api.ts`; `XS:` = `admin-ui/src/components/features/insights/ExamScope.tsx`
- `M:` = `upsc-document-generator/app/main.py`; `S:` = `backend/src/modules/sme/`
- Services: `S:services/sme-content-quality.service.ts` (CQ), `sme-release-health.service.ts` (RHS), `sme-notification-effectiveness.service.ts` (NES), `sme-onboarding-funnel.service.ts` (OFS), `sme-purchase-funnel.service.ts` (PFS)

---

## A.0 Data path (all five, verified)

| Page | UI → client | Portal route (forwards / drops) | Our route → service |
|---|---|---|---|
| question-quality | `getSmeQuestionQuality` API:3650 (has no `exam` param); page sends pool/subject/flag/minAttempts/days/includeInactive/sort/limit=200 (QQ:456–469) | M:5880–5905 `require_report_read`. Forwards pool, subject, topic, minAttempts(≥5), suspectRate, flag, sort, includeInactive, days(≤365), from, to, limit(≤200). **Declares no `exam` → DROPPED** | `S:controllers/sme-content.controller.ts:23–35` → CQ:89 |
| release-health | `getSmeReleaseHealth` API:3743; sends days or from/to, platform, exam (RH:110–121) | M:5908–5921: days(≤30), from, to, platform, exam. Drops nothing | `/sme/analytics/release-health` → RHS:84 |
| notification-fx | API:3831; sends days, maturationDays, minSent, poorReadRatePct, type, exam (NFX:78) | M:5924–5944: all 8 params; `minSent` ge=0 (ours clamps to ≥1, NES:95) | → NES:81 |
| onboarding-funnel | API:4027; sends days or from/to, exam (ONB:108–112) | M:5947–5959: days(≤365), from, to, exam | → OFS:475 |
| purchase-funnel | `getSmePurchaseFunnel` INS:46–52; sends `days` always and `exam` if set. Live call: `?days=30` | M:6193–6208: days (default 30, ≤90), exam. Declared above the catch-all | → PFS:92 |

- **Bug class 1 (query string replaced by `params`).** No caller builds a query onto the path, and `_sme_proxy` merges anyway (M:4357–4366). It does not apply.
- **Bug classes 2, 3, 7** do not apply: no `_sme_fetch_all`, no local filtering, explicit `Query()` params.
- **Errors.** Errors pass through as `HTTPException(status, detail)` (M:4383–4397). An unknown exam becomes `UnknownExamState` via `isUnknownExamError` (INS:83–86) on the four pages that send `exam`.
- **`unwrapExamRows` (INS:70–77) is not used by any of these five pages.**
  - None of the five routes switches shape under `?exam=`. Only Engagement uses the helper.
  - Release-health reads its top-level `examCoverage` directly (RH:355).
- **`ExamCoverageChip` (XS:40–66) appears on release-health only** (RH:355). Correct: it is the only one of the five that returns `examCoverage`.
- **Capture-plane facts checked on prod (E0):**
  - Raw `api_usage` min = 2026-10-03 20:01 UTC, so the 30-day purge runs and release-health's 30d preset is fully covered.
  - `auth_events` min = 2026-08-04 20:00 UTC (90-day purge runs).
  - Every surviving raw row carries `exam_id`: `since` = `raw_min`.
- **Internal accounts.** None of the five services excludes internal accounts. Purchase-funnel excludes only `orders.is_test` (PFS:140).

---

## 1. `/insights/question-quality`

### A — per-number table

Live call: `pool=both&minAttempts=5&days=90&includeInactive=false&sort=priority&limit=200`. Window = rolling 90×24h (`resolveIstWindow`, `sme-stats.util.ts:78–104`). The content pool filters `updated_at`; the simulation pool filters `simulation_attempts.startedAt`.

| # | Number on screen (UI line) | Plane | Our definition | Their transformation | Window/TZ | Exclusions | Verdict (SQL) | Sev |
|---|---|---|---|---|---|---|---|---|
| Q1 | **Eligible 978** (QQ:581) | A | `summary.eligible`: questions with ≥5 non-blank answers, active only (CQ:154, 245–252, 302, 441) | none | 90d rolling | **No internal exclusion** | **TRUE.** E1: 972 content + 6 simulation = **978**. Excluding internal accounts: 957 | 🟡 |
| Q2 | **Worse than chance 1 / Suspicious 4** (QQ:582–593) | A | Wilson 95% lower bound > 1−1/options → worse; > 0.6 → suspicious (CQ:494–501) | none | 90d | none | **TRUE** (E1: 1 / 4). The single "worse than chance" item (the Vedic-period simulation question, 48 answers) **drops to "suspicious" once the 1 internal answer is removed** (E1 real row: 0 / 5; E2a). | 🟡 |
| Q3 | **Learners misled 0.1** (QQ:594–598) | A | Σ max(0, LB − chance) × attempts over the flag-filtered set (CQ:171–174) | `.toFixed(1)` | 90d | none | **TRUE** (E1: 0.1). It is a fractional statistic. The name "learners" suggests people, but "0.1 learners" is not a count of anyone; see B. | 🟡 |
| Q4 | **Contradicts key 77** chip, and "· 77 contradict the key" (QQ:491, 576–580, 814–816) | A | `contradictsKey`: the most-picked wrong option beats the key (CQ:520–521) | **Counted client-side over the ≤200 returned items only** (QQ:491), not a server summary | 90d | none | **TRUE today by coincidence.** E1 counts 77 over all 978 eligible, and all 77 happen to sit in the top 200 by priority. The label does not say "in the top 200". When >200 rows tie at priority 0, it silently becomes a floor. MISLABELLED-latent | 🟡 |
| Q5 | **"200 questions in the queue"** (QQ:813) | A | `items` = `filtered.slice(0, limit)` (CQ:162) | `limit=200` hard-coded (QQ:468) | 90d | none | **MISLABELLED.** 978 are eligible and the queue shows the first 200 by priority. There is no "200 of 978". 195 of the 200 rows are flagged OK (snapshot). | 🟡 |
| Q6 | Row badges **"Contradicts key" + "OK" + "above the line"** on the same row (QQ:160–164, 290–293) | A | The flag uses the Wilson LB. "above the line" uses the raw wrong-rate > chance (QQ:125) | — | — | — | **MISLEADING.** **72 of the 77** contradicts-key rows carry flag "OK", typically at 5–9 answers. With so few answers Wilson cannot clear 60% even at 100% wrong. So one row says "answer key probably wrong", "OK" and "above the line" at once. | 🟠 |
| Q7 | Flag chips **"Suspicious" / "OK"** (QQ:615–626) | — | `flag` is a **severity floor** (CQ:159–160: `FLAG_RANK >= floor`) | label = exact flag name | — | — | **MISLABELLED.** "OK" returns every flag; "Suspicious" returns suspicious + worse. | 🟡 |
| Q8 | Header copy **"PYQ attempts are not included (they live in Neo4j)"** (QQ:43, 566, 799) | — | Pools: `content_questions` + `simulation_questions` only (dto `QUALITY_POOLS`) | — | — | — | **WRONG copy, and the biggest coverage gap.** PYQ answers persist in Postgres `pyq_attempts`, first row 2026-08-05 (E2c). In the same 90 days: **42,254 PYQ answers from 1,504 users; 1,619 PYQ questions have ≥5 users.** The content pool has 17,469 rows and the page shows 978 eligible. The most-answered MCQ bank is invisible. | 🟠 |
| Q9 | Question text ends **"……"** (QQ:306–307) | — | Backend already appends "…" when truncated (CQ:528–530) | The UI appends a second "…" | — | — | **Rendering bug** (2 instances in the snapshot) | ⚪ |
| Q10 | "(window 90d · min 5 attempts)" (QQ:737) | — | `meta.windowDays` | The UI prints the `days` it **sent**, not `meta.windowDays` | — | — | Equal today; would diverge only on clamping | ⚪ |

Note (not rendered): the content pool's `wrong` is the stored `is_correct` (CQ:295), but `isKey` / `contradictsKey` use the **current** `correct_answer` (CQ:490, 513). If a key is corrected after the fact, a row mixes the old key's wrongness with the new key's distribution. ⚪

### A.2 Exam scoping

- **No selector, nothing sent, and the proxy drops it** (M:5880–5905; API:3650 has no `exam`; C1 I2).
- Our side supports it (dto `:92–104`; CQ:94–96, 227–236, 360–371):
  - Content pool: `content_documents.exam_ids && [exam,'*']` (the CONTENT rule: `*` = every exam).
  - Simulation pool: through the linked simulations' `examIds`. An unlinked question is unreachable under a filter.
- No `examCoverage` chip, which is correct: there is no capture-plane source.
- Prod (E2e): answered content in the last 90 days is tagged `{upsc-cse}` 6,318 questions and `{appsc-group-1}` 536.
- **So today's queue silently mixes APPSC into one bank.** 🟠

---

## 2. `/insights/release-health`

### A — per-number table

Live call: `?days=14` → window 20 Oct – 3 Nov, rolling 14×24h (RHS:85). Plane B (raw `api_usage`) plus `auth_events` for client errors.

| # | Number on screen | Plane | Our definition | Their transformation | Window/TZ | Excl. | Verdict (SQL) | Sev |
|---|---|---|---|---|---|---|---|---|
| R1 | Per-row **Active users / Requests / Error rate (excl. paywall)**, e.g. 2.1 android 2,091 · 2,22,208 · 0.2%; 2.1 ios 1,301 · 1,49,850 · 0.1%; pre-1.7 10 · 747 · 0.7% (RH:183–212) | B | groupBy (app_version, platform, status); distinct users; (4xx+5xx−402)/(req−402) (RHS:107–125, 276–293) | `nf`/`pf`; rows re-sorted client-side (RH:154) | 14d rolling | **No internal exclusion** (2.1 android has 3 internal users, ios 3, web 2) | **TRUE.** E3 matches every row: 222,209 vs 222,208 (one request after capture), all user counts exact | ⚪ |
| R2 | **"incl. paywall X%"** sub-line + popover "Paywall 402s" (RH:205–210; RowDetailsPopover:38) | B | `status = 402` rows in `api_usage` (RHS:148) | — | 14d | none | **MISLABELLED (systematic undercount, same blind spot as S02 #10).** `UsageQuotaGuard` 402s never reach the interceptor. E4: **63 rows / 52 users** with a 402, all on `/pyq/:id/details` and `/quota/begin`, vs **737 users** who emitted `paywall_viewed` in the same 14 days. | 🟠 |
| R3 | **"Force-update number 0.3%"**, "**10 of 3,627** active users are on a build that predates the version header", and "This is the population `AppConfig.minAppVersion` would block on next launch if you raised it today" (RH:143–152, 386–416) | B | Client-side: Σ activeUsers of rows with `appVersion === null` ÷ Σ `platforms[].activeUsers` | UI arithmetic | 14d | none | **Numerator TRUE** (E4: 10 pre-header users; 1 of them also used a newer build). **Denominator inflated:** summing per-platform distincts gives 3,627, but the true distinct count is **3,581** (46 people on two platforms). It also includes web users, who are never force-updated. **The label is MISLABELLED:** "the population a raise would block" holds only for `minAppVersion = 1.7`. For the decision actually on the table (require 2.1), E4 shows **149 of 3,533 mobile users (4.2%)** last seen on a build below 2.1 (140 versioned + 9 pre-header). The page cannot answer that. | 🟠 |
| R4 | "Within-platform share" android 0.0% / ios 0.0% / web 0.0% / unknown 100.0% (RH:433–452) | B | `platforms[].preHeaderSharePct` (RHS:415) | — | 14d | — | **TRUE but uninformative by construction.** Pre-1.7 builds send no platform header either, so the unknown bucket is always 100%. The page itself says "Not the headline". See B. | ⚪ |
| R5 | **"vs. previous build" verdict** (Similar / Not enough data) (RH:214–226) | B + auth_events | Next-older build on the same platform; ±1 pt error rate or ±0.5 client errors per user; ≥200 req and ≥5 users on both (RHS:314–378) | `VerdictChip` | 14d | — | **TRUE mechanically.** Example: 2.1 android has 813 client errors / 2,091 users = 0.39, vs 2.0's 0. A delta of 0.39 < 0.5 gives "similar". | ⚪ |
| R6 | Popover **"Client errors (auth_events)" / "Client errors / active user"**; banner "Compare **crash rates** with All exams" (RowDetailsPopover:35–36; RH:360) | auth_events | `event_type='client_error'` grouped by version × platform (RHS:127–137) | — | 14d | none | **Count TRUE** (E3: 813 android 2.1, 795 of them without a user id; 76 ios). **"Crash" is MISLABELLED:** E4b shows **841 of 892** client errors in the window are `http_401` (token expiry/refresh). The next largest are network timeouts (~50). No crash capture exists. Every build since 1.7 emits `client_error` (first_seen per build, E4b), so the axis is comparable across builds. | 🟠 |
| R7 | Footer "Window 20 Oct – 3 Nov 2026 (14d)…" (RH:490–509) | — | `meta.from/to` rendered in IST | IST ✓ | — | — | **TRUE** | ⚪ |

### A.2 Exam scoping

- **Selector:** present (RH:328–337, "Exam (drill-down)"). Sent: RH:119. Forwarded: M:5920.
- **Coverage chip:** RH:355, shown only under an exam.
- **Client errors:** the app-wide flag is handled. `clientErrorsAppWide` (RH:136) drives the amber banner (RH:348–363), the "may be the inflated client-error axis" chip (RH:220–224) and the popover relabel.
- **NULL rule = REQUEST:** `api_usage.exam_id IS NULL` rows are **excluded**, never folded into upsc-cse (RHS:97–101). This is the opposite of the PERSON rule used by the other three exam-scoped pages.
- **Today:** every surviving raw row carries `exam_id` (E0), so `unknownRows` should be ~0 and "Per-exam data from …" equals the window start (S02 noted the same). ⚪

---

## 3. `/insights/notification-fx`

### A — per-number table

Live call: `days=60&maturationDays=3&minSent=50&poorReadRatePct=25`.
- Window 4 Sep 14:21 → 3 Nov 14:21 IST; matured = sent before 31 Oct 14:21 IST.
- Source: `notification_history` (person sends; topic broadcasts excluded).
- **The live response says `dailyRowCapHit: true`, complete from 2026-10-28 12:31 UTC.**

| # | Number on screen | Our definition | Their transformation | Excl. | Verdict (E5 = true values for the same window) | Sev |
|---|---|---|---|---|---|---|
| N1 | **Sent 1,79,133 / Read 6,131 / Types 7** (NFX:401–407) | DB-side `groupBy(type, isRead)`, exact (NES:120–129, 148–153) | `nfmt` | No internal exclusion (677 sends went to internal accounts) | **TRUE** (179,133 / 6,131 / 7) | ⚪ |
| N2 | **Matured sent 8,904** | Built from the **newest 20,000 rows only** (`NOTIF_DAILY_ROW_CAP`, NES:134–139, 157–164) | — | — | **WRONG.** True: **168,037.** The 20k pull starts at 28 Oct 18:01 IST, so the "matured" sample is the **~2.8 days** between then and 31 Oct 14:21, not 57 days. | 🔴 |
| N3 | **Matured read rate 2.4%** | maturedRead ÷ maturedSent on the capped sample (NES:256) | — | — | **WRONG. True 3.6%.** | 🔴 |
| N4 | **Wasted sends 8,691** | maturedSent − maturedRead, capped | — | — | **WRONG. True 162,019.** | 🔴 |
| N5 | **Pending 11,096** | Rows after `maturedBefore` (always inside the newest 20k) | — | — | **TRUE** (179,133 − 168,037 = 11,096) | ⚪ |
| N6 | **Kill list:** #1 pyq "6,642 wasted of 6,712 matured, 1%"; #2 doc 1,059/1,081, 2%; #3 my_activity 407/426, 4.5%; #4 premium 289/299, 3.3%; #5 daily_task 288/380, 24.2% (NFX:338–393) | `worstPerformers`, from capped `maturedSent`, sorted by wasted (NES:202–218) | — | — | **WRONG magnitudes, wrong order.** True: pyq **130,379 of 133,666, 2.5%** · doc **18,634/19,696, 5.4%** · premium **5,020/5,179, 3.1%** · my_activity **3,232/3,592, 10.0%** · daily_task **4,670/5,798, 19.5%**. premium and my_activity swap places (premium should be #3). The recommendation strings quote the wrong counts. | 🔴 |
| N7 | **By-type table** Matured sent / Matured read rate / Wasted / **Recipients** (NFX:428–484), e.g. pyq recipients 3,522; premium 621; daily_task 633; report 0% over 4; home 0% over 2 | Recipients = distinct users in the capped rows (NES:165) | — | — | **WRONG** except Sent/Read/Pending. True recipients: pyq **9,961**, premium **5,378**, daily_task **5,230**, doc 812, my_activity 1,896. report is **16.3% over 92** (UI 0% over 4); home is **50% over 14** (UI 0% over 2). | 🔴 |
| N8 | Footnote **"`byType` totals above are exact regardless"** (NFX:521–526) | — | — | — | **WRONG copy.** Only `sent`/`read` are exact. The caveat string the server ships next to it (rendered at NFX:563–571) says so correctly. | 🟠 |
| N9 | **Daily read rate** bars (NFX:487–531) | Per IST day from the capped rows (NES:169–178) | — | — | TRUE for the days it shows, which is **only 28 Oct onward** of a "60d" window. The header already calls it immature. | 🟡 |
| N10 | **Unit = `type`** ("Kill or heavily re-target 'premium'") | `notification_history.type` | — | — | **MISLABELLED.** `type` is a deep-link screen category, not a notification. E6b: premium = 3 rules (subscription_ended, subscription_ending, trial_ending); daily_task = streak_unfilled + trial_started; my_activity = streak_completed + weekly_progress. You cannot "stop sending" a type; the unit you can act on is the rule. `notification_history.rule_id` exists and is filled on 179,040 of 179,133 rows (E5). | 🟠 |
| N11 | Pointer box: **"Notifications sent by the rules engine carry a real per-notification tap … Notification Rules → a rule → History"** (NFX:149–156) | `notification_dispatch.tapped_at`, set by `POST /notifications/tapped` (`notification-preferences.controller.ts:59, 97–117`) | — | — | **WRONG in practice.** E6: **0 taps ever in 196,370 dispatch rows.** No client calls the endpoint: `rg` over `upsc_app` (HEAD `7645b63`, 2026-10-31) and `prepmonkey-web` finds no `notifications/tapped`, and the app never receives a `ruleId`. The "honest alternative" is a column that is always empty. | 🔴 |

### A.2 Exam scoping

- **Selector:** present ("Recipient's exam", NFX:322–327). Sent (NFX:78) and forwarded.
- **Rule = PERSON:** the recipient's `active_exam_id` as of now, with NULL ≡ upsc-cse (NES:110–114, `sme-exam-scope.util.ts examUserWhere`).
- Prod: 11,657 of 16,249 profiles are NULL, so "UPSC" means most users.
- No `examCoverage`, which is correct (full history).
- The banner (NFX:183–197) states the retro-attribution and "decide what to kill on All exams". ✔

---

## 4. `/insights/onboarding-funnel`

### A — per-number table

Live call: `?days=30`. The cohort is `user_auth.createdAt` in a rolling 30×24h window (OFS:489–504). Stages are current-state booleans evaluated now.

| # | Number on screen | Plane | Our definition | Verdict (E7) | Sev |
|---|---|---|---|---|---|
| O1 | **Signed up 4,370 · Phone verified 3,903 · Onboarding completed 2,983 · Psychometric resolved 3,009 · First action 1,562** (ONB leak diagram) | C/A | OFS:153–167, 176–296 | **TRUE, exact on every stage.** No internal-domain signups in this window (E7 internal row absent). | ⚪ |
| O2 | **End-to-end conversion 35.7%** "of 4,370 signups reach a genuine product action" (ONB:299–310) | A | Stage 5 = first row in the ENGAGED union (simulation_attempts, user_content, user_question_attempts, custom_tasks with no source_id, psychometric_test_results, user_document_progress) at or after signup (OFS:719–811) | **MISLABELLED.** (a) The UI's own breakdown shows 1,317 of the 1,562 (84.3%) are "took the psychometric test". (b) **PYQ practice is excluded** even though it persists in `pyq_attempts`: **262 of the 2,808 "lost" users did PYQ practice** (E7). Including it gives 1,824 = **41.7%**. | 🟠 |
| O3 | **"Biggest leak — First genuine product action −1,447, 48.1% of the previous stage"** (ONB:318–333) | A | `buildSummary` (OFS:299–324) | **Overstated by ≥262** for the same reason. | 🟠 |
| O4 | Phone verified "median 31s · p90 1.6m"; psychometric "median 5m · p90 14m, timed for 43.8% (completers only)" | C/A | Nearest-rank percentiles (OFS:92–137) | TRUE to definition (not re-derived in SQL) | ⚪ |
| O5 | **Login success rate 94%** "5,088 of 5,413 attempts succeeded"; raw event list login_attempt 5,413 · 0 ids … otp_verify_failed 401 · 242 ids (ONB:~700–760) | auth_events | `login_success ÷ login_attempt` counts (OFS:919–941). App-wide; app 2.0+ only (mobile emitters) | **TRUE as a floor.** "· 0 ids" beside 5,413 attempts (ONB:751) reads as "zero people". It means login events carry no identifier hash. | ⚪ |
| O6 | **Worst verification day** "16 Oct 72/89 (80.9%) … 13 otp_verify_failed vs median 12 — **a spike worth chasing**" | auth_events | `spiked = otpVerifyFailed > median` (OFS:409, 424–430) | **MISLABELLED.** Any day above the median is called a "spike". 13 vs 12 is not one. | 🟡 |
| O7 | Header "**Where the money leaks** — signup → phone → …" (ONB:126) | — | — | **Copy mislabel:** this funnel ends before any paywall. | ⚪ |
| O8 | "Cohort 04 Oct → 03 Nov" (ONB:339) | — | `fmtDay` slices the ISO **UTC** string (ONB:56–61) | Correct at 14:21 IST. It shows the previous day for captures made 00:00–05:30 IST. | ⚪ |
| O9 | 90d/365d presets with auth `byDay` | auth_events | Days before `firstCoveredDay` are null (OFS:363, 372). `firstCoveredDay` = capture start **2026-07-23** (OFS:826–835) | Purge leaves rows from **2026-08-04** only (E0). A 365d window shows **0 instead of null** for 23 Jul–4 Aug (bug class 6). The caveat mentions the purge in prose only. | 🟡 |

### A.2 Exam scoping

- **Selector:** present (ONB:182). Sent (ONB:111) and forwarded.
- **Cohort rule = PERSON:** current `active_exam_id`, NULL ≡ upsc-cse (OFS:494). The `authEvents` panel stays app-wide and is labelled "App-wide".
- `examSelected` sibling panel (ONB:274) and `RetroAttributedNote` (ONB:275–277). Rendered (C1 I16). ✔

---

## 5. `/insights/purchase-funnel`

### A — per-number table

Live call: `?days=30` (the UI always sends `days`; default 30, INS:48). The window is anchored at **IST midnight** 5 Oct → now (PFS:108–114) ✔.

| # | Number on screen | Plane | Our definition | Their transformation | Excl. | Verdict (E8/E9) | Sev |
|---|---|---|---|---|---|---|---|
| P1 | **Paywall viewed 1,220 · Upgrade tapped 231 · Checkout opened 61 · Checkout abandoned 53 · Purchase failed 8** (PF:194–287) | auth_events (client, app 2.0+) | Σ over IST days of `count(DISTINCT user_id)` per day (PFS:124–133, 166–183) | Labelled "totals are per-day users, summed" (PF:213–215, small grey) | NULL user_id excluded; **internal not excluded** (12 / 3 / 3 / 3 / 2 internal-day-users) | **TRUE** (E8 exact). Distinct people in the window are **1,087 / 196 / 48 / 41 / 6**. A reader saying "1,220 people saw the paywall" is wrong; the tooltip says so, the tiles do not. | 🟡 |
| P2 | **Paid orders 28** "server-recorded"; tooltip "Server truth: orders marked PAID (test orders excluded)" (PF:221–228) | C | `orders` status PAID, `is_test=false`, by IST creation day (PFS:135–143) | — | is_test only | **WRONG.** E9: **17 of the 28 are `payment_source = MANUAL`.** These are SME comp grants (`schema.prisma` enum comment; `payments.service.ts:1599`): 8 users, 11 orders on internal accounts, 16 to users who already had a paid order. **Only 11 are real payments** (Razorpay 4, Apple 7), and 2 of the Apple ones are renewals (`apple-payments.service.ts:1292` creates an order per DID_RENEW; Razorpay subscription cycles likewise, `razorpay-subscriptions.service.ts:737`). **New external buyers: 10.** | 🔴 |
| P3 | **Conversion 2.3%** "28 paid against 1,220 paywall views" (PF:246–263) | mixed | ordersPaid ÷ paywallViewed (PFS:47–50) | `convLabel` | — | **WRONG.** With real payments: 11 / 1,220 = **0.9%**. First-time buyers per distinct viewer: ~9–10 / 1,083 ≈ **0.9%**. | 🔴 |
| P4 | Gap **"−33, 54% lost, checkout → paid (not joined)"** (PF:204, 306–316) | mixed | checkoutOpened − ordersPaid | client | — | **WRONG** with P2. Real: 61 − 11 = −50. Of the 18 distinct buyers, 15 emitted paywall_viewed and 11 emitted checkout_opened (E9c). | 🟠 |
| P5a | **Checkout abandoned 53**, sub-label "pricing / friction" (PF:270–276) | auth_events (client) | Same per-day-distinct sum as P1 | Tooltip: "Opened the payment sheet and left it. If most losses are here, the checkout is the problem" | 3 internal users | **Count TRUE** (E8: 53 summed, **41 people**). **"pricing / friction" is unprovable:** no reason is captured (`error_code`, `message` and `metadata.reason` are all null on these rows, E10). The rows split into Razorpay (32 rows / 22 people) and Apple (25 / 21). 22 of the 41 also left a `subscriptions` row, i.e. a UPI AutoPay mandate was created and never approved: the customer-side abandonment the 2.2 work documented. 4 of the 41 paid for real later in the window. Customer-side placement is honest; the cause label is a guess. | 🟡 |
| P5b | **Purchase failed 8**, sub-label "**ours to fix**"; tooltip "**The payment itself failed.** Non-trivial counts are an engineering bug" (PF:277–286) | auth_events (client) | as P1 | — | 2 internal user-days | **Count TRUE** (8 summed; **6 people, 85 rows**). The tooltip **MISLABELS** it. All 85 rows carry `metadata.reason = create_subscription_failed` on Razorpay (E10b). **Every one of the 81 attributable rows followed a server `400` on `POST /api/v1/payments/subscriptions` within 2 minutes** (E10, 6 of 6 users). No payment was ever attempted: our server refused to create the mandate. That matches S03's finding that the 400s were the 15-minute "already active" block (`ABANDONED_PENDING_MS`), removed in backend `eafb96b`. So **"ours to fix" is honest** (it was our bug, now fixed), but **"the payment itself failed" is false.** Peaks: 2 Nov (41 rows, 3 people) and 19 Oct (20 rows, 1 person). | 🟠 |
| P6 | "Showing the last 30 days for all exams" (PF:159–166) | — | `data.days` echoed | — | — | **TRUE.** Renders the server's value ✔ | ⚪ |

### A.2 Exam scoping

- **Selector:** present (PF:91). Sent, and forwarded (M:6205).
- **Two attribution rules in one ratio:**
  - Steps = PERSON (current exam, NULL ≡ upsc-cse; PFS:117–119 via `examUserSql`).
  - `ordersPaid` = `orders.exam_id` exact (PFS:120). A NULL `exam_id` would be **excluded** under a filter. Prod: 0 NULLs among PAID orders in the window (E9b), so harmless today.
- "Directional" badge + paragraph (PF:167–181). ✔
- **No `examCoverage`, which is correct.** But MANUAL comp grants carry an exam (9 APPSC, 8 UPSC), so the per-exam paid count is inflated the same way. 🔴 (same root as P2)

### The two (three) opposite NULL rules across these pages

| Rule | NULL means | Pages |
|---|---|---|
| REQUEST (`api_usage.exam_id`) | excluded ("belongs to no exam") | release-health |
| PERSON (`user_profiles.active_exam_id`) | ≡ `upsc-cse` (72% of profiles are NULL) | notification-fx, onboarding cohort, purchase-funnel steps |
| CONTENT (`examIds` array) | `'*'` = every exam; untagged = unreachable under a filter | question-quality (backend only, not exposed) |
| ORDER (`orders.exam_id`) | excluded under a filter | purchase-funnel ordersPaid |

The pages state their own rule (banners, the "Directional" badge). None of them explains why the same "UPSC" filter means different populations on adjacent pages.

---

## B. MEANING

> **Owner ruling 2026-11-03 (verbatim):** "The numbers are there BUT those numbers are not explaining anything — same on the aspirant profile screen. No need to drastically change stuff; compact the details or combine and show. I don't need to see the endpoints a user is hitting on the portal, it explains nothing. Instead explain in simple terms: some X metric done on this data, or 'he stayed X mins', or 'he has been using this many times'. I may be grossly oversimplifying — the portal might already do some things well and corrections may not be needed at all. Just analyze and judge."

### Shareholder questions

| Question | Answered here? | What is missing | Already persisted? |
|---|---|---|---|
| (1) Feature usage, with repeat rate | Not these pages | — | — |
| (2) Per-exam and per-language funnels | **Partially.** Onboarding and purchase funnels take `exam` (retro-attributed). No language anywhere. | A language split. A per-exam view without retro-attribution. | Exam: `active_exam_id`, `orders.exam_id`. Language: `user_profiles.ui_language`. Both persist, so a language split is derivable today. A history of exam switches **needs new persistence** (flag only). |
| (3) Where buyers stop in checkout | **Partially, and the last step is wrong.** The step counts are right. "Paid" counts comp grants and renewals (P2). "Purchase failed" is really "our server refused to start the mandate" (P5b). Abandoned has no reason (P5a). | New-vs-renewal-vs-comp split. The plan each step was for. | `orders.payment_source`, `apple_original_transaction_id`, `receipt LIKE 'sub_%'`. `auth_events.metadata` carries `{planId, source}` on funnel events (`schema.prisma` AuthEvent comment). `purchase_failed.error_code`. **All persisted.** |
| (4) Question quality | **Partially.** Content + simulation banks only. **The PYQ bank (2.4× the answers) is excluded.** No exam scoping in the portal. | PYQ pool; exam selector | `pyq_attempts(question_id, user_id, selected_option, is_correct, attempt_number, created_at)` persisted since 2026-08-05. Exam scoping exists on our side. |
| (5) Notification outcomes | **Not reliably.** Rates come from ~2.8 days (N2–N7). The unit is the screen type, not the rule. Taps are never captured. | Full-window rates per rule. A real open/tap signal. | Full window: `notification_history.isRead/createdAt/rule_id`, computable today. Tap: column exists (`notification_dispatch.tapped_at`) but **no client emits it**, so it needs client emission (flag). Feed-open time: `notification_history.read_at` persists. |
| (6) Support view of one user | Not these pages | — | — |

### B.0 The reference: `/insights/purchase-funnel` → **KEEP** (owner: "very excellent")

**Why it passes the plain-language test.** A non-engineer can read it top to bottom and say: *"In the last 30 days ~1,100 people saw a paywall, ~200 tapped upgrade, ~50 opened checkout, and about a dozen paid. Most were lost right after seeing the paywall, which is a copy or price question."*

Seven properties make that possible. These are the copyable pattern:

1. **A one-line subtitle that states the question** (PF:87): "Where the money stops: paywall seen → upgrade tapped → checkout opened → paid, per IST day. The losses between steps are the point."
2. **A scope line that echoes what the server actually used** (PF:159–166): "Showing the last 30 days for all exams", from `data.days`/`data.exam`, not from the request.
3. **Step tiles with one plain count each** (PF:194–228). No route names, no event names, no rates inside the tiles.
4. **Every loss between steps is a labelled object with its likely cause** (PF:200–205, 306–316), e.g. "−989 · 81% lost · interest — paywall copy or price". The reader learns where people stop and what to try.
5. **One headline ratio with its caveat beside it** (PF:246–268): "Paid ÷ paywall viewed 2.3%", "28 paid against 1,220 paywall views", and "a rising rate can mean more phones on app 2.0, not a better paywall".
6. **Failures split into the customer's and ours** (PF:270–287): "Checkout abandoned · pricing / friction" vs "Purchase failed · ours to fix". That sorts the reader's next action (growth vs engineering).
7. **One per-day chart; numbers and caveats folded away** (PF:184–186): exam selector + window pills + Refresh as the only controls.

**Keep the shape. Fix four facts** (A P1–P5; none change the layout):
- (a) "Paid" must exclude MANUAL staff comps (17 of 28) and split new vs renewal. Real figure: 11 payments, ~10 new buyers. This moves the headline from 2.3% to ~0.9%.
- (b) Tiles should count distinct people in the window (1,087 / 196 / 48 / 41 / 6), not the sum of per-day distincts (1,220 / 231 / 61 / 53 / 8). Then the drill-down list length equals the tile.
- (c) The "Purchase failed" tooltip should say "our server refused to start the purchase", not "the payment itself failed".
- (d) "pricing / friction" should read "left the payment sheet (no reason captured)".

#### B.0a Drill-down sidebar per step

Owner ruling: a simple list. One row per user: **user · kind of payment tried (Apple / Razorpay one-time / UPI AutoPay mandate / staff comp) · date**, each linking to `/app-users/:id`. Nothing else.

Rules common to every step:
- **Name** comes from `user_auth.name` joined on the user id.
- **Date** is the event's IST date, `(created_at + 330 min)::date`.
- **Dedupe.** The tile today is `Σ over IST days of count(DISTINCT user_id)` (PFS:124–133, 166–183), so a person seen on 3 days is counted 3 times. A 3-column list must show **one row per person, latest date**, or its length will not match the tile (it would match only after fix (b)).
- **Today's route returns no user lists.** `GET /sme/analytics/purchase-funnel` takes only `days` and `exam` (dto `sme-purchase-funnel.dto.ts`, PFS:92–205), and no param returns ids.

| Step (tile) | Source of the user ids today | Retention | Count rule today | "Kind of payment" column: which field, buildable? | Can our backend return the list today? |
|---|---|---|---|---|---|
| Paywall viewed 1,220 | `auth_events.user_id` where `event_type='paywall_viewed'`. 23 of 1,522 rows have NULL user and cannot be listed | **90 d** (nightly purge); emitted by app 2.0+ only | per-IST-day distinct, summed (1,087 people) | **Not applicable**: nobody has picked a payment yet. `metadata` holds only `{screen, source}` (E10). Show "—" or the paywall screen. Buildable as a 2-column list. | **No.** Needs a per-step user-list endpoint (flag only) |
| Upgrade tapped 231 | `auth_events.user_id`, `upgrade_tapped` | 90 d, app 2.0+ | same (196 people) | **Partly.** 158 of 383 rows carry `metadata.gateway` (apple / razorpay) + `plan` + `productId`; 225 rows carry only `{feature, reason, source}` (E11). **Apple vs Razorpay is buildable where the gateway is present. Razorpay one-time vs UPI AutoPay mandate is not in the event**: it needs a time-join to `subscriptions.created_at` (mandate) or `orders` with a non-`sub_` receipt (one-time) for the same user. | **No.** New endpoint (flag) |
| Checkout opened 61 | `auth_events.user_id`, `checkout_opened` | 90 d, app 2.0+ | same (48 people) | **Gateway on every row** (`metadata.gateway`, `plan`, `productId`). Apple: yes. Razorpay one-time vs mandate: **only by time-join to `subscriptions`/`orders`** (same as above). | **No.** New endpoint (flag) |
| Checkout abandoned 53 | `auth_events.user_id`, `checkout_abandoned` | 90 d, app 2.0+ | same (41 people) | Gateway on every row: Razorpay 22 people, Apple 21. **22 of the 41 have a `subscriptions` row** (`user_id, provider, status, plan_type, exam_id, created_at, first_charge_granted_at IS NULL`), so "UPI AutoPay mandate (never approved)" can be shown for those by join. | **No.** New endpoint (flag) |
| Purchase failed 8 | `auth_events.user_id`, `purchase_failed` | 90 d, app 2.0+ | same (6 people, 85 rows) | Every row: `gateway=razorpay`, `reason=create_subscription_failed`, so **"UPI AutoPay mandate (server refused)"**, buildable directly from metadata. | **No.** New endpoint (flag) |
| Paid 28 | `orders."userId"` (`status=PAID`, `is_test=false`, `created_at`) | **permanent** | **count of orders**, not people (28 orders / 18 people) | **Fully buildable today.** `orders.payment_source`: APPLE / RAZORPAY / **MANUAL = staff comp**. Within RAZORPAY: `receipt LIKE 'sub_%'` (or `razorpay_order_id LIKE 'sub_%'`) = UPI AutoPay mandate charge, otherwise one-time order. Apple renewal = an earlier PAID order with the same `apple_original_transaction_id`. In this window: Razorpay 4 (all mandate), Apple 7, MANUAL 17 (E11b). | **Yes, approximately.** `GET /sme/transactions?orderStatus=PAID&source=…&exam=…&startDate=…&endDate=…` (`sme-transaction.dto.ts:41–119`) already returns these orders with user ids. `source` takes one value, so the funnel's "all real payments" needs two calls (APPLE, RAZORPAY) or a new filter. |

**Exam per row.** `metadata.productId` names the exam actually being bought on every gateway-bearing event (`…premium.monthly` = UPSC, `…appscgroup1.pre…` = APPSC). That is **better than the current retro-attributed `active_exam_id`**: it is the exam on the paywall that was tapped. It could replace the "Directional" caveat for steps 2–5 (computable today; step 1 has no productId).

**Is the customer-vs-ours split honest?**
- **"Checkout abandoned → customer (pricing/friction)": honest placement, guessed cause.** No reason is persisted. The Razorpay half is largely the UPI AutoPay mandate-approval drop-off already proven customer-side in the 2.2 work. The Apple half is a StoreKit sheet cancel. Neither proves "pricing".
- **"Purchase failed → ours to fix": honest, and it was literally our bug.** 100% of the window's purchase failures are server refusals of `POST /payments/subscriptions`, i.e. the now-removed 15-minute "already active" block (S03; `eafb96b`). It is **not** a payment failure. Once that fix is live, this tile should drop to ~0, and any new non-zero is real.
- The split misses one thing that is ours: **orders/payments that reached the gateway and FAILED server-side** (`orders.status = FAILED`, `payments.status = FAILED`, permanent). They appear on neither tile. Flag only.

### B.1 The other four pages, card by card, each proposal in the purchase-funnel shape

Template per page: **subtitle (the question) → scope line → step/count tiles with plain counts → labelled losses with likely cause → one headline ratio + caveat → customer-vs-ours split where meaningful → one per-day chart → exam selector + window pills (+ custom range, see C.1) + Refresh.** Every count tile is tappable. The drill-down columns are listed per page.

#### Question quality → COMPACT into the template

| Template slot | Proposal | Source / computable today? |
|---|---|---|
| Subtitle | "Which questions have a broken key or a trap option, and how many learners hit them." | — |
| Scope line | "Last 90 days · all exams · content + mock-test + **PYQ** questions with ≥5 answers". **Add the exam selector** (the proxy currently drops `exam`, M:5880–5905). | Exam: our side ready. PYQ pool: `pyq_attempts` (first attempt per user), persisted |
| Step tiles | **Answered by ≥5 learners 978 → Most learners disagree with the key 77 → Confidently worse than guessing 1.** Replaces the 5 chips; "Learners misled 0.1" is dropped (RAW). | Yes (E1). Count the contradicts tile server-side over all eligible, not over the 200 returned (Q4) |
| Labelled losses | Not a funnel; label each tile with its action: "77 · check the key first" / "1 · rewrite or retire" | — |
| Headline ratio + caveat | "8% of well-answered questions have a disputed key", with "a question needs ~12 answers before 'worse than guessing' can be proven; below that we only say 'disputed'" | Yes |
| Customer vs ours | **Ours:** key disputed / no explanation attached. **Not a defect:** hard but fair (suspicious only) | Yes (`contradictsKey`, `hasExplanation`, flag) |
| Per-day chart | **None.** It is a work queue. The queue rows themselves are KEEP (the server `reason` sentence is already the plain sentence), with the flag chip fixed (Q6) | — |
| Drill-down per tile | Tile → the question list (already the queue). Per question → users who answered: `user_question_attempts.user_id, selected_option, updated_at` (permanent, latest answer only) / `simulation_answers` via `simulation_attempts."userId"` / `pyq_attempts.user_id` | No user-list route today; needs a new endpoint (flag) |

#### Release health → COMPACT into the template

| Template slot | Proposal | Source / computable today? |
|---|---|---|
| Subtitle | "Is the newest build healthy, and who would be locked out if we force an update?" | — |
| Scope line | "Last 14 days · all platforms · all exams" (exists, RH:490–495) | — |
| Step tiles | **Phone users seen 3,533 → on the newest build (2.1) 3,384 → on an older build 149 (of them 9 too old to report a version).** Replaces "Force-update number 0.3%" and drops "Within-platform share" (RAW) | Yes: latest `app_version` per user from `api_usage`, mobile only, distinct across platforms (E4) |
| Labelled losses | "−149 · 4.2% · would see a blocking update screen if 2.1 were required" | Yes |
| Headline ratio + caveat | "Server errors on 2.1: 2 in 1,000 requests, same as 2.0", with "paywall refusals are not errors; requests blocked before reaching the server are not counted" | Yes (`errorRateExcludingPaywallPct`) |
| Customer vs ours | **Ours:** 5xx and network errors. **Not a fault:** expired sign-ins (`http_401`, 841 of 892 "client errors") and paywall 402s. Relabel; stop saying "crash" | Yes (`auth_events.error_code`). Real crashes **need new persistence** (flag) |
| Per-day chart | Users per build per IST day (adoption curve). The service has no series today | Computable from `api_usage(created_at, app_version, user_id)`, 30-day raw cap; endpoint change (flag) |
| Version table | KEEP one row per build: people · error rate · verdict. Requests / "incl. paywall" move into the tooltip | — |
| Drill-down per tile | "149 on an older build" → users: `api_usage.user_id`, last `app_version`, last `created_at` | **30 d raw** only; no user-list route (flag) |

#### Notification FX → COMPACT into the template

| Template slot | Proposal | Source / computable today? |
|---|---|---|
| Subtitle | "Which automatic notifications bring people back, and which ones should we stop?" | — |
| Scope line | "Sent 4 Sep – 31 Oct (old enough to judge) · all exams" | — |
| Step tiles | **Sent 168,037 → person opened the notification list afterwards 6,018 (3.6%) → tapped the notification —** (not recorded). Totals over the **whole** window, not the 20k-row sample (N2–N4) | Sent/opened: yes, `notification_history`. Tapped: **client never reports it** (N11). Flag |
| Labelled losses | Per **rule** (not type): "PYQ reminder · 130,379 of 133,666 never followed by a visit · likely: too frequent, or the same people every day" | `notification_history.rule_id` → `notification_rule.name`, permanent |
| Headline ratio + caveat | "3.6% of notifications were followed by a visit to the notification list", with "this measures coming back, not tapping this message; a low number is the trustworthy direction" (the current `isReadMeaning`, in one line) | Yes |
| Customer vs ours | **Ours:** too many sends per person (sends ÷ recipients: pyq 142,520 / 9,961 ≈ 14 each in 60 days) and sends to opted-out or uninstalled devices. **The person's:** ignored | Sends per person: yes. Delivery failure: not persisted (flag) |
| Per-day chart | Sends per IST day per rule, over the whole window (replaces the immature read-rate bars) | Yes; the service currently caps at 20k rows (endpoint change, flag) |
| Kill cards | KEEP the card format (the sentence per card is right) with full-window numbers per rule | — |
| Drill-down per tile | Recipients of a rule: `notification_history."userId", createdAt, isRead` | **Permanent** (not purged); no user-list route (flag) |

#### Onboarding funnel → mostly KEEP (it is already the template's ancestor), relabel

| Template slot | Proposal | Source / computable today? |
|---|---|---|
| Subtitle | "Where new sign-ups stop before they start studying." Replace "Where the money leaks" (O7) | — |
| Scope line | "People who signed up 4 Oct – 3 Nov · all exams" (exists, ONB:339) | — |
| Step tiles | KEEP **4,370 → 3,903 → 2,983 → 3,009 → studied something 1,824**. Stage 5 adds `pyq_attempts`; the psychometric test is shown as its own step, not as "studied" (O2) | Yes (E7) |
| Labelled losses | KEEP the coral losses; add the cause line like PF: "−467 · phone OTP"; "−920 · onboarding form"; "studied: most only took the psychometric test" | — |
| Headline ratio + caveat | "42% of sign-ups studied something", with "the newest sign-ups have not had time yet" (exists) | Yes |
| Customer vs ours | **Ours:** OTP send/verify failures (`otp_send_failed` server-side, complete). **The person's:** left the form / never came back | Yes (`auth_events`, 90 d) |
| Per-day chart | KEEP the by-signup-day table, or turn it into one chart: sign-ups vs phone-verified per IST day | Yes |
| OTP/login panel | COMPACT to one sentence (B before): "94% of sign-ins on app 2.0+ succeeded; 242 phone numbers failed an OTP at least once." Drop the raw event list and the "not derivable" paragraph | Yes |
| Drill-down per tile | Each stage → users: `user_auth.id, name, createdAt, phone_verified`; `user_profiles.onboardingCompleted/psychometric*`; first action from the ENGAGED tables + `pyq_attempts` | **Permanent** (auth_events part 90 d); no user-list route (flag) |


---

## C. GAP — documented params/fields vs proxy vs UI

| Item (doc) | Status |
|---|---|
| QQ `?exam=` (SME_INSIGHTS_API §1 L179, controller :27–29) | **absent.** Proxy drops it; API client has no param (C1 I2) |
| QQ `topic`, `suspectRate`, `from`/`to` | **proxied-only** (C1 I7) |
| QQ `meta.exam`, `meta.simulationAttemptCapHit`, `meta.windowDays`, `summary.ok` | **absent.** Not rendered (grep count 0 in QQ) |
| RH `platforms[].worstVersion*` | **absent** (grep 0) |
| RH `adoptionSharePct`, `serverErrorRatePct`, `clientErrors*`, `errors4xx/5xx` | rendered in popover only |
| NFX `from`/`to` | **proxied-only.** The UI only sends `days` |
| NFX `meta.typeFilter`, `meta.examFilter` | absent (UI uses its own state) |
| NFX `notification_dispatch.tapped/tapRate` (decoder §5.1) | linked page exists; **data is always 0** |
| ONB, PF fields | rendered (C1 I16–I19, U22) |
| PF as a panel on `/engagement` (FRONTEND_GUIDE L387) | **absent.** Standalone page (C1 G10) |
| Stale UI copy | QQ "PYQ … live in Neo4j" (QQ:43, 566, 799); NFX:525 "exact regardless"; RH:360 "crash rates"; RH:411–416 "the population minAppVersion would block"; ONB:126 "Where the money leaks"; ONB:424-ish "a spike worth chasing" (server string, OFS:428) |


### C.1 Standard controls per route: Exam · window pills · custom date range · Refresh

None of the five routes takes `startDate`/`endDate`; those names belong to the `/sme/analytics/{metric}` catch-all DTO. Four of the five accept **`from`/`to`**: an ISO datetime, or a bare `YYYY-MM-DD` read as an **IST** day (`parseIstBoundary`, `sme-stats.util.ts:55–71`). On those four, the max span is enforced even with an explicit `from`/`to` (`resolveIstWindow` :78–104 silently clips `from`, and `meta.windowDays` reports the clip).

| Route | Range params accepted today | DTO cap | Proxy forwards | UI today | Source retention (label a range beyond it) |
|---|---|---|---|---|---|
| question-quality | `days`, `from`, `to` | 365 d (`QUALITY_WINDOW_MAX_DAYS`) | days (≤365), from, to (M:5880–5905); **`exam` dropped** | Window select 30–365 (QQ:52, 674–684) · **no custom range** · Refresh ✓ · **no exam** | `user_question_attempts`, `simulation_*`, `pyq_attempts`: **permanent** (but only the *latest* answer per user is kept for content questions, and `pyq_attempts` starts 2026-08-05) |
| release-health | `days`, `from`, `to`, `platform`, `exam` | 30 d (`RELEASE_WINDOW_MAX_DAYS` = raw retention) | all (M:5908–5921; `days` le=30) | Pills 7/14/30 · **custom range ✓** (RH:286–310) · Refresh ✓ · exam ✓ | `api_usage` raw **30 d** (prod floor 2026-10-03 20:01 UTC). An older `from` sets `meta.windowExceedsApiUsageRetention` and the UI shows a note (RH:424–430) ✓. `auth_events` (client errors) 90 d |
| notification-effectiveness | `days`, `from`, `to`, `type`, `exam`, `maturationDays`, `minSent`, `poorReadRatePct` | 120 d (`NOTIF_WINDOW_MAX_DAYS`) | all (M:5924–5944; days le=120) | Pills 14/30/60/90/120 · **no custom range** (from/to proxied-only) · Refresh ✓ · exam ✓ | `notification_history` **permanent** (no purge in `api-usage-rollup.service.ts`). No retention label needed. The 20k-row cap (N2) is the real limit |
| onboarding-funnel | `days`, `from`, `to`, `exam` | 365 d (`FUNNEL_WINDOW_MAX_DAYS`) | all (M:5947–5959) | Pills 7/14/30/90/365 · **custom range ✓** (ONB:203–221) · Refresh ✓ · exam ✓ | Cohort tables **permanent**. `auth_events` **90 d** (prod floor 2026-08-04): the response labels days before the *capture start* (2026-07-23) as null, but **not** days lost to the purge (O9) |
| purchase-funnel | **`days` and `exam` only; no `from`/`to`.** The window is always "the last N IST days ending today" (PFS:108–114) | 90 d (`PURCHASE_FUNNEL_MAX_DAYS`) | days (le=90), exam (M:6193–6208) | Pills 7/14/30/60/90 · **no custom range** · Refresh ✓ · exam ✓ | Steps: `auth_events` **90 d**, and app 2.0+ only (≥2026-09-17). Paid: `orders` **permanent**. **A custom range needs a backend param** (flag only). A range older than 90 d must be labelled "steps not recorded", never drawn as zeros next to real paid orders |

---

## D. Contradictions (report, do not paper over)

1. **QQ copy vs data.** QQ:43, 566, 799 say PYQ attempts "live in Neo4j". Postgres `pyq_attempts` holds 42,254 rows in the last 90 days, first row 2026-08-05 (E2c). C1 did not catch this one.
2. **NFX UI vs server caveat vs service.**
   - NFX:525 says "`byType` totals above are exact regardless".
   - The server caveat (NES:246) and SME_INSIGHTS_API §3 Caveats (L778–781) correctly say only `sent`/`read` are exact.
   - The service itself builds maturedSent / maturedRead / wastedSends / recipients / `worstPerformers` from the capped rows (NES:157–167, 189, 202–218).
   - With the UI's 60d default and ~2.4k sends/day, the cap **always** bites. Every kill-list number on the live page is a ~2.8-day sample.
3. **Tap signal.**
   - Claimed: SME_METRICS_DECODER §5.1 (L280–293) and NFX:149–156 present `notification_dispatch.tapped_at` as the honest per-notification signal, which "app 2.0 is the first build expected to call".
   - Data: 0 taps in 196,370 rows (E6). No client caller exists.
   - The decoder also names the route `POST /notification-preferences/tapped`, but the controller is `@Controller('notifications')`, so the route is `POST /notifications/tapped` (`notification-preferences.controller.ts:59, 97`).
4. **"ordersPaid is server truth" (excluding test orders only).**
   - Where it is claimed: the `PFS:189` caveat, the PF:222 tooltip, SME_USAGE_ANALYTICS §4c L305, FRONTEND_GUIDE L329 and SME_INSIGHTS_API §5.
   - What it actually includes: MANUAL comp grants (`payments.service.ts:1599`) and renewal orders (`apple-payments.service.ts:1292`; `razorpay-subscriptions.service.ts:737`). 17 of 28 in this window are MANUAL (E9).
5. **The release-health headline's meaning.**
   - SME_INSIGHTS_API §2 "How to use" (L585–607) and RH:411–416 say the pre-header share is "the population `minAppVersion` would block if you raised it today".
   - That is true only for a raise to 1.7. A raise to the current 2.1 would block 149 / 3,533 mobile users (E4).
6. **"Crash".**
   - Claimed: SME_METRICS_DECODER L252 ("The crash-proxy"), L199/L366 ("crash rates"), SME_INSIGHTS_API L413/578, and RH:360 all call `client_error` a crash proxy.
   - Data: 94% of it is `http_401` (E4b).
7. **Onboarding capture floor.** OFS:826–835 and the `interpretation` string use the capture start 2026-07-23 as the first covered day. The 90-day purge means the oldest `auth_events` row is 2026-08-04 (E0), so long windows show 0, not null, for the gap.
8. **Inherited from C1, unchanged.**
   - QQ `?exam=`: doc §0.1 / §7 say "still open" while our code supports it.
   - The PF DTO typo "numerator denominator" (`sme-purchase-funnel.dto.ts:43`).
   - Purchase funnel is "a panel" in the docs; it is a page in the UI.
   - "Current store build is 2.0".
9. **Brief vs SHARED_RULES.** SHARED_RULES says "No prod/staging SQL in this pass". This lane brief permits read-only prod SQL. I followed the brief.
10. **The "Purchase failed" tooltip vs the data.**
    - PF:280 and SME_INSIGHTS_API §5 "The action that follows" both describe `purchase_failed` as "the payment itself failed" and call a non-trivial count "an engineering bug".
    - Every `purchase_failed` in the window is `create_subscription_failed` after a server 400 on `POST /payments/subscriptions` (E10). The cause was our 15-minute "already active" block (S03, `eafb96b`). No payment was attempted.
    - "Engineering bug" was right; "payment failed" was not.
    - The "pricing / friction" label on abandoned (PF:275) has no persisted reason behind it.
11. **The owner's reference screen has two wrong facts in its own example.** SHARED_RULES "B reference" quotes "Paid ÷ paywall viewed 2.3%". That figure includes 17 MANUAL staff comps. Real payments give ~0.9% (P2/P3). The layout stays the reference; the number does not.

---

## E. SQL run (production, read-only) and results

All queries are in `/private/tmp/claude-502/-Users-yashwanth-Desktop-projects/b0942abe-5468-460e-92b6-db352e6a3606/scratchpad/s06sql/`.

The runner was `npx ts-node -T scripts/staging-db.ts --prod --file …`, output piped through `rg -v 'eyJ|rzp_|sk-'`. Internal accounts = `user_auth.email ILIKE '%@stanzasoft.%' OR '%@prepmonkey.%' OR '%@designasylum.in'`. Columns were checked against `information_schema.columns` first.

- **E0** (inline): `min(created_at)` api_usage = 2026-10-03 20:01:55 UTC; auth_events = 2026-08-04 20:00:02; first `exam_id` row = raw_min.
- **E1 `q1_qq.sql`** (QQ summary, both pools, Wilson in SQL, 90d to 08:50 UTC, minAttempts 5, active only):

  | scope | pool | eligible | worse | suspicious | contradicts (all eligible) | learners misled |
  |---|---|---|---|---|---|---|
  | portal | content | 972 | 0 | 3 | 74 | 0.0 |
  | portal | simulation | 6 | 1 | 1 | 3 | 0.1 |
  | portal | total | **978** | **1** | **4** | **77** | **0.1** |
  | real (excl. internal) | total | 957 | 0 | 5 | 72 | 0.0 |

- **E2 `q2_qq_detail.sql`**
  - (a) The Vedic-period question exists as 5 near-duplicate simulation questions; the scored one has 48 answers, 1 from an internal user.
  - (b) 130 submitted simulation attempts in 90 days (2,000 cap not hit).
  - (c) `pyq_attempts`: first row 2026-08-05 08:50; 90 days = 42,254 rows / 1,504 users / 3,565 questions; **1,619 questions with ≥5 distinct users**.
  - (d) `user_question_attempts` 90 days = 17,469 rows / 753 users / 6,854 questions.
  - (e) Answered content by tag: `{upsc-cse}` 6,318, `{appsc-group-1}` 536.
- **E3 `q3_rh.sql`** (release-health per version × platform, 14d to 08:51 UTC): every UI row is reproduced.
  - 2.1 android 222,209 req / 2,091 users (3 internal) / 456 4xx / 3 5xx / 36 × 402 / 0.2% / 813 client errors (795 without a user).
  - 2.1 ios 149,856 / 1,301 / 0.1% / 76 client errors.
  - 2.1.0 web 7,577 / 85 / 0.3% / 2.
  - 2.0 android 4,041 / 70 / 0.3% · 2.0 ios 2,225 / 31 · 1.9 a/i 485/22, 982/19 · 1.8 a/i 132/3, 174/6 · 1.7 a/i 3/1, 169/3.
  - NULL/NULL 747 req / 10 users / 0.7%.
- **E4 `q4_rh_head.sql`:**
  - distinct users 3,581, vs sum of per-platform distincts 3,627
  - pre-header users 10 (1 also on a newer build, 0 internal)
  - mobile users by latest build: 3,533; latest build < 2.1 = 140; latest = pre-header 9
- **E4b** (inline):
  - `client_error` first_seen per build: 1.7 android 2026-08-04 … 2.1 ios 2026-10-14 (all builds emit).
  - Last 14 days by `error_code`: http_401 841, ClosedByteChannel 11, ConnectTimeout 10, SocketTimeout 9, SSLHandshake 8, other ≤5.
  - 402 rows 63 / users 52 (routes `/api/v1/pyq/:id/details`, `/api/v1/quota/begin`), vs `paywall_viewed` 984 events / 737 users.
- **E5 `q5_nfx.sql`** (60d, matured before 2026-10-31 08:51:06 UTC, no row cap):

  | type | sent | read | matured sent | matured rate | wasted | recipients | to internal | by rules |
  |---|---|---|---|---|---|---|---|---|
  | all | 179,133 | 6,131 | 168,037 | 3.6% | 162,019 | 10,436 | 677 | 179,040 |
  | pyq | 142,520 | 3,335 | 133,666 | 2.5% | 130,379 | 9,961 | 383 | 142,520 |
  | doc | 20,864 | 1,071 | 19,696 | 5.4% | 18,634 | 812 | 233 | 20,864 |
  | daily_task | 6,132 | 1,170 | 5,798 | 19.5% | 4,670 | 5,230 | 15 | 6,132 |
  | premium | 5,501 | 163 | 5,179 | 3.1% | 5,020 | 5,378 | 8 | 5,501 |
  | my_activity | 4,009 | 370 | 3,592 | 10.0% | 3,232 | 1,896 | 38 | 4,009 |
  | report | 92 | 15 | 92 | 16.3% | 77 | 29 | 0 | 0 |
  | home | 15 | 7 | 14 | 50.0% | 7 | 14 | 0 | 14 |

  The newest-20k pull starts at 2026-10-28 12:31:07.744 UTC (matches the live caveat).
- **E6 `q6_tap.sql`:**
  - Dispatch rows joined to matured history, 60d, `is_test=false`: 129,141 rows, **0 tapped**.
  - All time: `tapped_at` min NULL, **0 taps / 196,370 dispatch rows**.
  - **E6b** (inline), rules per type: daily_task 2 (streak_unfilled, trial_started) · doc 1 (continue_learning) · home 1 (payment_success) · my_activity 2 (streak_completed, weekly_progress) · premium 3 (subscription_ended, subscription_ending, trial_ending) · pyq 1 (pyq_reminder) · report 0.
- **E7 `q7_onb.sql`** (30d cohort to 08:51 UTC): signups 4,370 · phone 3,903 · onboarding 2,983 · psychometric 3,009 · first action 1,562 (all exact).
  - Internal signups 0.
  - No first action but did PYQ: **262**; first action incl. PYQ: 1,824.
  - (No first action but any api_usage: 2,802 of 2,808. Not discriminating, because onboarding requests are authenticated.)
- **E8 `q8_pf.sql`** (IST 5 Oct 00:00 → 3 Nov 08:51:30 UTC):

  | step | summed per-day distinct | summed excl. internal | distinct in window | distinct excl. internal |
  |---|---|---|---|---|
  | paywall_viewed | **1,220** | 1,208 | 1,087 | 1,083 |
  | upgrade_tapped | **231** | 228 | 196 | 193 |
  | checkout_opened | **61** | 58 | 48 | 45 |
  | checkout_abandoned | **53** | 50 | 41 | 38 |
  | purchase_failed | **8** | 6 | 6 | 4 |

- **E9 `q9_orders.sql`:**
  - PAID, non-test orders in the window: total **28**.
    - RAZORPAY 4 (4 subscription orders, 4 new external buyers).
    - APPLE 7 (2 renewals, 5 new external buyers).
    - **MANUAL 17** (11 on internal accounts, 16 to users with an earlier paid order, 8 users, 1 new external).
  - New external buyers overall: 10.
  - (b) exam_id on PAID orders: RAZORPAY appsc 1 / upsc 3; APPLE appsc 4 / upsc 3; MANUAL appsc 9 / upsc 8; 0 NULL. MANUAL in window: 17 orders, ₹14,584 total, 0 zero-amount, 8 users.
  - (c) 18 distinct buyers: 15 emitted paywall_viewed, 11 emitted checkout_opened.
- **Profiles** (inline): `active_exam_id` NULL 11,657 · upsc-cse 3,390 · appsc-group-1 1,202.

- **E10 `q10_fail_split.sql`** (same purchase-funnel window):

  | event | rows | rows without user | people | rows with error_code / message | metadata keys |
  |---|---|---|---|---|---|
  | paywall_viewed | 1,522 | 23 | 1,087 | 0 / 0 | screen+source |
  | upgrade_tapped | 383 | 4 | 196 | 0 / 0 | feature+reason+source, or gateway+plan+productId+source |
  | purchase_failed | 85 | 4 | 6 | 0 / 0 | gateway+plan+productId+reason+source |
  | checkout_opened | 73 | 0 | 48 | 0 / 0 | gateway+plan+productId+source |
  | checkout_abandoned | 57 | 0 | 41 | 0 / 0 | gateway+plan+productId+source |

  - `purchase_failed`: 6 people / 81 attributable rows. **6 of 6 people and 81 of 81 rows follow an `api_usage` 4xx on a `/payments/` route within 2 min**, and the only such route+status is `/api/v1/payments/subscriptions 400`.
  - Abandoned: 41 people. 4 also made a real (non-MANUAL) payment in the window; **22 created a `subscriptions` row**; 3 internal.
  - **E10b** (inline): `purchase_failed` = 85 × `reason=create_subscription_failed`, `gateway=razorpay`, `source=client`. `checkout_abandoned` = razorpay 32 rows / 22 people, apple 25 / 21, reason NULL. `purchase_failed` by IST day: 10-05 9, 10-07 1, 10-19 20, 10-20 1, 10-30 9, 11-02 41 (3 people), 11-03 4.
- **E11** (inline), `metadata` gateway/plan/productId on steps 2–5:
  - `upgrade_tapped`: 225 rows without a gateway, 158 with (razorpay 122, apple 36).
  - `checkout_opened`/`abandoned`/`purchase_failed`: gateway on every row.
  - productId names the exam (`…premium.*` = UPSC, `…appscgroup1.pre…` = APPSC, one `upsc-cse.monthly`).
  - **E11b**: PAID orders in the window, by kind: RAZORPAY 4 (all `receipt LIKE 'sub_%'`, MONTHLY), APPLE 7 (MONTHLY), MANUAL 17 (16 MONTHLY, 1 ANNUAL).

**Processes:** none left running. Every `ts-node` invocation ran in the foreground and exited. No background pids were started.
**Files written:** this report, plus the SQL files in `scratchpad/s06sql/` (q1–q10), and splice parts in `scratchpad/s06parts/` (the pre-amendment copy is `S06_before.md`).
