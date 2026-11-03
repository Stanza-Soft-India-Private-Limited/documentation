# C1 — Docs ledger: analytics / insights / activity-trail docs (audit C, agent 1 of 3)

Read-only pass, 2026-11-03. Backend `4ea16f2` (HEAD, 2026-11-03). Portal `7b00daf` (2026-10-27, local clone; **not fetched** per rules, so it may be behind origin).
Paths: `B:` = `/Users/yashwanth/Desktop/projects/backend/`, `M:` = `portal/app/main.py`, `UI:` = `portal/admin-ui/src/`.
No post-2026-09-21 commits touch any of the analytics/insights/activity services, controllers or DTOs (`git log --since=2026-09-22` over all 10 files is empty), so the backend half of every doc here is still the `f6329e6` build.

Verdict legend: **RENDERED** (UI sends it / shows it) · **PROXIED-ONLY** (reaches main.py, never the UI) · **BACKEND-ONLY** (we have it, portal never calls it) · **ABSENT** (doc promises it, backend lacks it) · **CONTRADICTED** (doc and code disagree; both quoted).

---

## SME_USAGE_ANALYTICS.md (493 lines; field-level contract for `/sme/analytics/*`)

| # | Doc promise (doc line) | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| U1 | 14 base routes summary…notifications (§1–§4) | `sme-analytics.controller.ts:63-165` | catch-all `M:6211`, allowlist `M:6186-6190` | `api.ts:2792-2807`, all 14 called from `EngagementAnalytics.tsx:375-401` | RENDERED |
| U2 | `?days` default 30, max 120 (L12) | `sme-analytics.dto.ts:28-34` (`@Max(120)`), service clamp `sme-analytics.service.ts:109` | `M:6214` `le=120` | `RANGES=[7,30,60,90]` `EngagementAnalytics.tsx:363` | RENDERED |
| U3 | `?startDate=&endDate=` (L12) | dto `:36-44`, `window()` `:108-114` | forwarded `M:6215-6216,6235` | never sent (`uaQuery` callers pass only days/exam, `api.ts:2792-2807`) | PROXIED-ONLY. Also: `paywallHits`/`heatmap` ignore `endDate`; they use only `since` (`service:452-460,477-487`) |
| U4 | `?exam=` on every route, unknown slug = 400 (§0b rule 1) | `resolveExam` `service:61-65`; `@ApiExamQuery` on all 18 routes | forwarded `M:6222,6237` | `useExamParam` + `ExamSelector` `EngagementAnalytics.tsx:369,450`; 400 canary `:430-437` via `isUnknownExamError` `insights.api.ts:83-86` | RENDERED |
| U5 | Four routes change shape → `{data, examCoverage}` (L132-135) | `dau :202-208`, `paywallHits :462-467`, `heatmap :489-492`, `endpoints :511-517` | passthrough | `ExamWrapped<T>` `api.ts:2755`; normalised once `EngagementAnalytics.tsx:409-413` with `unwrapExamRows` `insights.api.ts:70-77` | RENDERED |
| U6 | `summary` gains TOP-LEVEL `examCoverage` (L113,147) | `service:150-158` | passthrough | `UsageSummary.examCoverage?` `api.ts:2759`; chip `EngagementAnalytics.tsx:469` | RENDERED |
| U7 | "since <date>" label mandatory where `examCoverage` appears (L98-101) | — | — | `ExamCoverageChip` `components/features/insights/ExamScope.tsx:40-66` ("Per-exam data from {since}", null → "Not recorded for this exam yet"); placed on summary `:469`, dau `:502-505`, paywall `:596`, heatmap `:709-713`, endpoints `:804` | RENDERED (all 5 Engagement panels + release-health) |
| U8 | `since` ISO instant vs IST `YYYY-MM-DD` on dau/endpoints (L95) | `apiUsageCoverage :74-83` (ISO), `apiUsageDailyCoverage :86-94` (date) | — | `formatCoverageSince` handles both, anchors bare date at IST noon `insights.api.ts:60-67` | RENDERED |
| U9 | `unknownRows` = this response's discard count, never reused (L88-91,96) | same predicate as route `:74-94` | — | per-panel chip shows "· N excluded" + tooltip "This count is for this panel only" `ExamScope.tsx:43-45,62-64` | RENDERED |
| U10 | `since:null` ⇒ "not recorded yet", not "no traffic" (L84-86) | — | — | `notRecorded()` empty-state copy `EngagementAnalytics.tsx:424-425,508,597,716,805` | RENDERED |
| U11 | Never compare per-exam vs pre-deploy all-exam (L99) | — | — | `RetroAttributedNote` on pulse `:479-485` ("must not be read as a ratio"); DAU boundary marker `:139,155-158,509` | RENDERED |
| U12 | PERSON NULL ≡ upsc-cse; REQUEST NULL excluded (§0b rule 2) | `sme-exam-scope.util.ts`; `SME_EXAM_QUERY_DESCRIPTION` `dto:14-18` | — | users panel "never-picked users count as UPSC" `:897`; chip copy "belong to no exam, not to UPSC" `ExamScope.tsx:44` | RENDERED |
| U13 | Retro-attribution label on PERSON routes (§0b rule 4) | — | — | `RetroAttributedNote` on signups `:520`, features `:535`, retention `:636`, churn `:651`, platforms `:738`, notifications `:798`, users `:897` | RENDERED |
| U14 | `users` — `isPremium` "is still the person-level mirror — it is **not** re-scoped to the exam" (L159) | **CONTRADICTED**: `users()` switches `isPremium` to a live `user_exam_entitlements` row in that exam under `?exam=` — `service:256-307` (EXISTS probe `:268-276`, switch `:302-304`), commit `f6329e6` 2026-09-21; controller summary says so `controller:88` | passthrough | UI follows the stale doc: crown tooltip "Premium on ANY exam (paid or trial) — not necessarily the exam selected above" `EngagementAnalytics.tsx:915-917` | CONTRADICTED (doc + UI copy wrong under `?exam=`) |
| U15 | `users` row `actions30d` (L158) | key name `actions30d` but computed over the **request window** (`since = window(q)`, `service:257,287`) — 7/60/90 when the selector says so | — | column header "Actions 30d" + tooltip "in the last 30 days" `:907` while `days` is sent `:395` | MISLABELLED (doc + UI) |
| U16 | `premium-engagement` meaning switches under `?exam=` (L181-193) | `service:411-440` (entitlement ids `:419-425`) | — | tier relabelled "Entitled to X / Not entitled" vs "Premium (any purchase) / Free" `:551-569` | RENDERED. Card title says "(last 30 days)" `:544,546` but backend uses `days` window `:412,430` → MISLABELLED at 7/60/90 |
| U17 | `paywall-hits`/`heatmap` ~30-day raw window (L196,203,402) | raw `api_usage`, purge 30d | — | ">30d" note "(last 30d only — window not applied)" `:587-593,700-707` | RENDERED |
| U18 | `content` per-exam sums above total (L210) | `service:548-579` | — | note `:771-775` | RENDERED. `topDocuments[].subject` is fetched but never rendered (`:755-760`) |
| U19 | `platforms` = token owner's exam, not installs (L208) | `service:526-545` | — | `:738` | RENDERED |
| U20 | `notifications` flat; prefer notification-effectiveness (L212,279-281) | `service:584-595` | — | caption + link to `/insights/notification-fx?exam=` `:794-797` | RENDERED |
| U21 | §4b release-health / notification-effectiveness / onboarding-funnel exam semantics | see SME_INSIGHTS_API rows I2–I4 | | | RENDERED (see below) |
| U22 | §4c purchase-funnel: days 1..90, `conversionPct` null not 0, >100 allowed, totals summed, exam caveat | `sme-purchase-funnel.service.ts:47,101,140,162,193,201`; dto `:43` | `M:6193-6208` (`le=90`, both params) | `InsightPurchaseFunnel.tsx`: presets `:29` 7/14/30/60/90, echoes `data.days` `:161-164`, "Directional" badge `:172`, null→"no views — no rate" `:247,440,519-521`, gap in line `:347-357`, >100 annotated `:260,418-419`, caveats `:185,472-480` | RENDERED |
| U23 | §5 capture / rollup / purge bounds | `ApiUsageInterceptor`, `api-usage-rollup.service.ts` | — | — | n/a (informational) |
| U24 | §7 "No SME endpoint reads [auth_events] yet (that's a later module)" (L411-412) | **STALE**: release-health, onboarding-funnel, purchase-funnel and the activity trail all read `auth_events` | | | CONTRADICTED (stale passage) |
| U25 | §4b "Time-to-read is derivable only from 2026-07-23 onward" (L248) | `timeToRead.derivable: false` (SME_INSIGHTS_API L686-706; UI `InsightNotificationFx.tsx:534-542`) | | | CONTRADICTED (doc vs sibling doc and code) |
| U26 | "Current store build is 2.0" (L228-229) | stale: app 2.1 is in the field (paid order on 2.1 on 2026-10-31) and 2.2 ships 2026-11-03 | | | stale passage |

**Verdict: implemented (~21/24 contract rows rendered).** Most consequential misses: (1) U14 — the doc (and the portal copy built from it) says `isPremium` is not exam-scoped; the code made it exam-scoped the same day; (2) U15/U16 — "30d" labels on numbers that follow the 7/60/90 selector; (3) U3 — `startDate/endDate` never reach the UI.

---

## SME_INSIGHTS_API.md (1415 lines; five decision endpoints)

| # | Doc promise (doc line) | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| I1 | Five distinct screens, open-and-refresh, never polling (L7-52) | — | — | 5 routes `App.tsx:163-167`; `staleTime`/`refetchOnWindowFocus:false` e.g. `InsightPurchaseFunnel.tsx:71-77`, `InsightNotificationFx.tsx:80-81` | RENDERED |
| I2 | **question-quality `?exam=`** — §1 query table says supported, 400 on unknown, `meta.exam` echoed (L179,222-262) **but** §0.1 (L97-99) and §7 (L1395) say "does not / still open" | **Works**: `sme-content-quality.dto.ts:92-104`, `meta.exam` `:258-259`, controller description `sme-content.controller.ts:27-29`; landed in `bfae389` 2026-09-21 | **DROPPED**: `M:5880-5905` declares no `exam` and does not forward it | `getSmeQuestionQuality` has no `exam` param `api.ts:3650-3663`; `SmeQuestionQualityMeta` has no `exam` `api.ts:3637-3642`; page has no `ExamSelector` (`InsightQuestionQuality.tsx`, absent from the `ExamSelector` importer list) | BACKEND-ONLY; doc self-CONTRADICTED (L98/L1395 vs L179) |
| I3 | `contradictsKey` first-class chip + default sort (L226-228,277-282) | `sme-content-quality.service.ts:520,554,560` | passthrough | badge `InsightQuestionQuality.tsx:170,290`, filter + contradicts-first sort `:491-506` | RENDERED |
| I4 | `minAttempts` control defaulted low (L331-337) | dto 5–1000 | `M:5885` `ge=5` | default 5 `InsightQuestionQuality.tsx:429`, control `:664-667` | RENDERED |
| I5 | `reason` verbatim, `optionDistribution` bars, "mark handled" in portal state (L284-295) | service | — | `:189,346-348`, localStorage handled set `:46,91-114` | RENDERED |
| I6 | Empty ≠ clean; PYQ attempts (Neo4j) excluded — say on screen (L339-342) | — | — | `:565-566,799` | RENDERED |
| I7 | question-quality `topic`, `suspectRate`, `from`/`to` params (L180-186) | dto | forwarded `M:5883-5892` | not sent `:457-468` (only pool/subject/flag/minAttempts/days/includeInactive/sort/limit) | PROXIED-ONLY |
| I8 | release-health `days` max 30, `platform` (L374-387) | `sme-release-health.dto.ts` | `M:5910` `le=30`, platform/exam forwarded `:5920` | presets 7/14/30 `InsightReleaseHealth.tsx:46`, clamp 30 `:92`, platform select `:315` | RENDERED |
| I9 | `meta.clientErrorsExamScoped:false` under `?exam=` — render it or don't offer the filter (L390-413,528-538) | release-health service | passthrough | `clientErrorsAppWide` `InsightReleaseHealth.tsx:136`, copy "client errors per active user is inflated… a worse verdict can come from that axis alone" `:359`, flag on `worse` verdicts `:220`, popover `:233` | RENDERED |
| I10 | release-health top-level `examCoverage` + since label (L447-550) | | | type `api.ts:3739`; chip `InsightReleaseHealth.tsx:355`; pre-header share caveat `:420` | RENDERED |
| I11 | `app_version IS NULL` = pre-1.7 row, never dropped; `windowExceedsApiUsageRetention` footnote (L356-380) | | | `:177`, `:424-428` | RENDERED |
| I12 | notification-effectiveness: `isRead` is a bulk feed-open flag; `meta.isReadMeaning` must travel (L653-685) | service | `M:5924-5944` (all 8 params) | `isReadMeaning` banner directly under headline `InsightNotificationFx.tsx:127-132` | RENDERED |
| I13 | `isRead` vs `tapped` — doc says "push delivery is not tracked at all" and has **no mention** of the rules-engine per-dispatch tap | `notification_dispatch.tapped_at` (`schema.prisma:2174`, `3c8bc41` 2026-08-26), surfaced by `GET /sme/notification-rules/:id/analytics` (`notification-rule.service.ts:428`) | — | UI already says it: "Notifications sent by the rules engine carry a real per-notification tap instead… Notification Rules → a rule → History" `InsightNotificationFx.tsx:141-160`; `tapped`/`tapRate` typed `api.ts:2467-2468` | Doc STALE (SME_METRICS_DECODER §5.1 has it; this doc and SME_USAGE_ANALYTICS §4b do not). UI correct |
| I14 | `?exam=` on notification-effectiveness = recipient's exam; don't rank kill-list per exam (L728-748) | | forwarded | "Decide what to kill on All exams" `InsightNotificationFx.tsx:192` | RENDERED |
| I15 | `timeToRead.derivable:false`, never synthesised (L686-706) | | | `TimeToReadPanel` `:534-542` | RENDERED |
| I16 | onboarding-funnel `examSelected` sibling block (L1046-1099) | onboarding service | `M:5947-5959` | `ExamSelectedPanel` `InsightOnboardingFunnel.tsx:274,847-905` (cohortSignups/windowSignups, explicitlySelected(Pct), nullCount/nullBucketedAs, byExam, note) | RENDERED |
| I17 | `authEvents` stays app-wide under `?exam=` — label it (L1033-1044) | | | `AuthEventsPanel examScoped` `:281`, "App-wide" chip `:698`, by-day column "Auth (app-wide)" `:934` | RENDERED |
| I18 | stage-5 `detail.bySource` is an ARRAY (decoder §8 #4 said it rendered `[object Object]`/`NaN%`) | | | array-aware `toSourceEntries` `InsightOnboardingFunnel.tsx:525-546`; type comment `api.ts:3905-3907` | RENDERED (fixed) |
| I19 | purchase-funnel (§5) | see U22 | | | RENDERED |
| I20 | §0 "Errors are the Nest default `{ statusCode, message }`" (L84-86) | errors are `{success:false,message,…}` from the exception filter (AUDIT §2.1; SME_ACTIVITY_TRAIL_API L79+) | | | CONTRADICTED (stale passage) |
| I21 | Version ladder "Current store build is 2.0" (L358-362) | stale, see U26 | | | stale passage |

**Verdict: partial (~17/19 rendered).** Missing: (1) I2 — question-quality has had `?exam=` on our side since `bfae389`, but the proxy drops it and the page has no selector, so a mixed-exam bank is still reported as one queue; (2) I7 — topic / suspectRate / custom range not exposed. The doc itself states both "supported" (§1) and "not supported" (§0.1, §7).

---

## SME_METRICS_DECODER.md (375 lines; "one line per number on every SME screen")

This doc describes screens rather than promising routes, so the rows check its descriptions against today's code.

| # | Decoder claim (line) | Code today | Verdict |
|---|---|---|---|
| D1 | `/analytics` "Portal fetches at most **6,000 users** (100/page × 60 pages)" (§1 L71; §8 #2 L353) | `SME_FETCH_MAX_PAGES = 400` → 40,000 rows (`M:5520`, AUDIT §1.3). Failed later pages are still dropped silently (`M:5557-5564`) | STALE (cap); still-open part = dropped pages |
| D2 | Apple "Events recorded" inflated: proxy drops `?source=APPLE` (§1 L105; §8 #1 L351) | fixed: `_sme_fetch_all("/sme/webhooks", max_pages=30, params={"source":"APPLE"})` `M:5644-5646` + merge in `_sme_proxy` `M:4356-4364` | STALE (closed) |
| D3 | Engagement Pulse labels "DAU (24h)", "active N", "WAU/MAU" (§2 L124-132) | UI labels "Studied · 24h", "opened the app: N", "Studied · 7d", "Studied MAU · 30d", "Opened-app MAU · 30d" `EngagementAnalytics.tsx:473-477` | STALE labels (the numbers match) |
| D4 | User explorer: UI tooltip says "screen opens, taps…" — wrong, fix it (§2 L186) | fixed: tooltip lists the six tables and says opens/taps are not actions `:907` | STALE (closed); §8 #7 already lists it closed, so §2 contradicts §8 |
| D5 | "Actions 30d = … in the last 30 days" (§2 L186) | computed over `days` window (`service:257,287`) | CONTRADICTED (see U15) |
| D6 | "Premium until — NULL for trial users" (§2 L188) | true in code (trials set `trialEndsAt`, not `premiumExpiresAt`), but UI header tooltip says "Person-level premium expiry across ANY exam **(paid or trial)**" `:907` and the crown says "(paid or trial)" `:916`, while the Premium-vs-free card on the same page says "active trials count as *free*" `:556-557` | UI copy contradicts itself and the decoder |
| D7 | Monetization "premium/free" split on `premium_expires_at > now` (§2 L146) | true without `?exam=`; with it, live entitlement (`service:418-426`) — decoder §2 never says so; §0.1 row 2b does | partial (§2 table stale for exam) |
| D8 | Exam-scoped keys table (§2 L190-204) | all present and rendered (U6–U10, I9, I16, I17) | RENDERED |
| D9 | Purchase funnel table (§2 L206-216) | U22 | RENDERED |
| D10 | Question quality default minAttempts 20 (§3 L222) | API default 20 (true); portal sends 5 `InsightQuestionQuality.tsx:429` | true for API; the screen differs (by design) |
| D11 | §5.1 rule-engine `tapped`/`tapRate` | present (I13) | RENDERED |
| D12 | §8 #3 "paywall hits still offers a 7/30/60/90 selector over ~30 days" | still offered (`RANGES` `:363`) but now labelled "(last 30d only — window not applied)" `:587-593` | partially closed (labelled, not removed) |
| D13 | §8 #4 stage-5 `[object Object]`/`NaN%` | fixed (I18) | STALE (closed) |
| D14 | §8 header "re-verified against portal build v2.32.0 on 2026-07-29" | portal is at v2.44 (`7b00daf` merge "content-reviewer-v2.44") | stale passage |
| D15 | §4 release-health / §6 onboarding tables | match code (I8–I11, I16–I18) | RENDERED |
| D16 | §7 App Users "Platform filter … portal's local filter compares against provider/subscriptionSource" | not re-checked in this pass (App Users roster belongs to another doc group) | not verified |

**Verdict: partial — about half its "watch out" claims are now stale.** The §8 "Still open" list has four items: #1, #2's 6,000 cap and #4 are closed in code; #3 is only labelled. Two claims are actively wrong today: D5 (window) and D6 (the UI's trial wording).

---

## SME_ANALYTICS_FRONTEND_GUIDE.md (439 lines; portal brief: console IA, exam selector, coverage contract)

| # | Guide requirement (line) | UI today | Verdict |
|---|---|---|---|
| G1 | ONE global exam selector, top of page, scopes every panel (L44-49, L383, L435) | Engagement: one selector, all panels `EngagementAnalytics.tsx:450` + `opt()` keys `:373`. Pages with `ExamSelector` + `exam` passed: `/engagement`, `/analytics` (`Analytics.tsx:196,235`), release-health `:330`, notification-fx `:326`, onboarding-funnel `:182`, purchase-funnel `:91`. **Missing**: `/insights/question-quality` (no selector; proxy drops `exam`), `/` Dashboard (`AppOpsDashboard.tsx:31`, `AnalyticsDashboard.tsx:12`, `ContentDashboards.tsx:80` all call `getSmeAnalytics()` with no exam) | partial — per-page, not global |
| G2 | Selection in URL (L56) | URL on Engagement (`useExamParam` `ExamScope.tsx:14`) and all four insight pages; **`/analytics` keeps it in `useState('')`** `Analytics.tsx:196`; sidebar links carry no `?exam=` (`Sidebar.tsx:119-123`), so the scope is lost on every page change | partial |
| G3 | Options from `GET /sme/exams`, "All exams" default sends no param (L50-55) | `ExamSelector` → `GET /sme/exams` (AUDIT §1.2 /exams); `ex = exam \|\| undefined` `:371` | RENDERED |
| G4 | 400 → "that exam no longer exists" (L57-59, L398) | `UnknownExamState` `EngagementAnalytics.tsx:430-437`; purchase-funnel `:80` | RENDERED |
| G5 | Normalise the four wrapped shapes once (L63-74) | `unwrapExamRows` `:409-413` | RENDERED |
| G6 | "since <date>" chip mandatory; handle ISO + IST date; `unknownRows` behind it; null ⇒ not recorded (L76-117, L409) | U7–U10 | RENDERED |
| G7 | Clip the series at `since` or annotate the boundary (L114-115, L397) | DAU chart marks the first non-null `activeDau` with "per-exam data starts — earlier is unknown, not zero" `:139,155-158,509`; heatmap note `:712` | RENDERED |
| G8 | Retro-attribution vs coverage-floor copy (L119-128) | U13 | RENDERED |
| G9 | Premium tier relabel under `?exam=` (L251-257) | `:567-569` | RENDERED |
| G10 | **"Money — premium-engagement + paywall-hits side by side, with the `purchase-funnel` panel beneath them"** (L387) | **No purchase-funnel panel on `/engagement`.** It is a separate page `/insights/purchase-funnel` (`App.tsx:167`, `Sidebar.tsx:123`). The only link from Engagement appears under the paywall card **only when an exam is selected** `EngagementAnalytics.tsx:599` | partial (separate page; no link at All exams) |
| G11 | Clamped window note for heatmap/paywall at 60/90 (L395) | `:587-593,700-707` | RENDERED |
| G12 | purchase-funnel echoes clamped `days`; render that (L395) | `InsightPurchaseFunnel.tsx:161-164` | RENDERED |
| G13 | Coverage contract — summary 6 fields + examCoverage (L411) | `:473-477,469` | RENDERED |
| G14 | dau (L412) | `DauChart` `:121-190` | RENDERED |
| G15 | features humanised (L413) | `FEATURE_LABELS` `:532` | RENDERED |
| G16 | users: all 8 row fields + envelope (L414) | id (link), email, name, phone, premiumExpiresAt, lastSeen, actions30d, isPremium (crown) `:907-926` | RENDERED; copy wrong for `isPremium` under exam (U14) and for the "30d" window (U15) |
| G17 | signups, retention (L415-416) | `:514-521,609-640` | RENDERED |
| G18 | churn-risk "sortable, exportable" (L238, L417) | list + pager `:652-689`; no sort, no export | partial |
| G19 | premium-engagement 5 fields (L418) | `:562-578` | RENDERED |
| G20 | purchase-funnel all fields (L419) | U22 (5 steps incl. `checkoutAbandoned`/`purchaseFailed` — 11 refs) | RENDERED |
| G21 | paywall-hits, heatmap, endpoints (L420-422) | `:596-599,709-718,804-815` | RENDERED. Endpoint routes are shown raw (`/api/v1/...` prefix kept; `humanRoute = (r) => r` `:74`), so "humanize route templates" (L438) is not done |
| G22 | platforms (L423) | `:722-742` | RENDERED |
| G23 | content: topDocuments `title`·`subject`·`readers`; simulation 3 fields (L424) | title + readers only `:755-760`; **`subject` dropped**; simulation shown `:765-766` | partial |
| G24 | notifications (L425) | `:786-790` | RENDERED |
| G25 | "Never render a null as 0" — `activeDau`, `conversionPct`, `examCoverage.since`, `maturedReadRatePct` (L436) | activeDau gap `:171-176`; conversionPct U22; since U10; `maturedReadRatePct: number \| null` `api.ts:3765` | RENDERED |
| G26 | Exam-scoped stat row must not show active as % of engaged (L156-160) | pulse renders them as separate tiles, note says not a ratio `:482` | RENDERED |

**Verdict: implemented, with gaps (~21/26 rendered, 5 partial).** Most consequential: (1) G1/G2 — the "one global selector" is per-page and resets on every navigation; `/analytics` doesn't keep it in the URL, and question-quality and the `/` dashboard have no selector at all; (2) G10 — the purchase funnel is not on the console and has no link from it at All exams; (3) G16 copy (U14/U15).

---

## SME_ACTIVITY_TRAIL_API.md (1080 lines; snapshot / timeline / incidents / notes / support-code)

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| T1 | 5 routes | `sme-activity.controller.ts:29-106` | `M:7002-7075` | `api.ts:3420,3473,3528,3550,3575` | RENDERED |
| T2 | snapshot `?days` | dto `:103`, clamp `sme-activity.service.ts:221` | `M:7005` `le=90` | `AppUserDetail.tsx:122` | RENDERED |
| T3 | timeline `sources, from, to, days, limit, cursor` | dto `:113-146` | all 6 forwarded `M:7024-7030` | `api.ts:3470`; `useTrail.ts:132` | RENDERED |
| T4 | **timeline item `exam`** (null except orders / api_usage / feedback_reports) | dto `:260-268` | passthrough | `SmeTimelineItem.exam` `api.ts:3445`; `ExamCell` `ActivityItem.tsx:23,163`; focus dimming `:38,108`; sessions grouping `session-utils.ts:192` | RENDERED |
| T5 | **incidents `?exam=`** narrows the api_usage half only; pre-deploy rows excluded, not absent | dto `:187-205`, controller `@ApiQuery` `:89` | forwarded `M:7042,7053` | `IncidentsPanel.tsx:110-114` (selector "Any exam" `:135-137`), caveat copy `:157-163` | RENDERED |
| T6 | **snapshot `quota.examId`** (also on `available:false`) | dto `:348-353` | | `SnapshotHeader.tsx:412-427` | RENDERED |
| T7 | **snapshot `entitlement.entitlements[]` + `activeExamId`** (null ≠ UPSC) | dto `:376-420` | | `SnapshotHeader.tsx:56-70,292-309` (empty array handled), `api.ts:3391-3395` | RENDERED |
| T8 | `recentOrders[].examId` | dto `:434-435` | | `UNRESOLVED_EXAM`/`reconcileHint` in `SnapshotHeader.tsx` (rg hit) | RENDERED (spot-checked) |
| T9 | `retention` object (api_usage 30-day purge) — render `note` | dto `:239-245` | | snapshot `SnapshotHeader.tsx:683-689`; timeline `Timeline.tsx:312-347`; incidents `IncidentsPanel.tsx:176-178`; sessions `SessionsView.tsx:239-283` | RENDERED |
| T10 | Quota today-only, Redis TTL; `available:false` ≠ 0 | | | "Today (IST)… resets at midnight IST — no history is kept" `SnapshotHeader.tsx:421-445` | RENDERED |
| T11 | `permissionGranted:null` ≠ denied | | | "unknown (older build)" `SnapshotHeader.tsx:498` | RENDERED |
| T12 | notes `note, category, author` | dto `:213-225` | proxy whitelists exactly these 3 `M:7065` | `NotesComposer.tsx` | RENDERED |
| T13 | support-code lookup (+ `activeExamId`) | controller `:29-45` | `M:7069-7075` (path-quoted) | `Header.tsx:497`; type `api.ts:3558-3575` | RENDERED |
| T14 | Windows: timeline max 90 vs api_usage 30-day purge | `WINDOW_MAX_DAYS` | `le=90` | presets 7/30/90 `Timeline.tsx:26`; the 90 preset reaches past retention but `retention.note` renders (T9) | RENDERED (honestly labelled) |
| T15 | Stored route keeps `/api/v1` prefix — strip it yourself | — | — | not checked on the trail; Engagement endpoints table shows raw routes (G21) | not verified |

**Verdict: implemented (~14/14 checked rows rendered).** All four exam additions from WHAT_CHANGED §4 are on screen.

---

## SME_USER_DETAIL_PAGE_GUIDE.md (456 lines; proposed IA for `/app-users/:id`)

| # | Proposal (line) | `/app-users/:id` today | Verdict |
|---|---|---|---|
| P1 | Dedicated route `/app-users/:id`, linkable (L45-48) | `pages/AppUserDetail.tsx`; churn list links to it `EngagementAnalytics.tsx:660-662` | RENDERED |
| P2 | Two entry points: roster row + header support-code lookup (L50-54) | roster drawer keeps "Full profile" link (`AppUsers.tsx:962-973`); `Header.tsx:497` | RENDERED |
| P3 | IA order Snapshot → Incidents → Timeline → Notes (L62-76) | `AppUserDetail.tsx:200-287`: `SnapshotHeader` → `ExamFocusBar` → `IncidentsPanel` → `Timeline` (with a Sessions/Events view toggle) → `NotesComposer` | RENDERED (+ two additions: exam focus bar, sessions view) |
| P4 | Three GETs fired independently (L78-80) | separate `useQuery`s (`AppUserDetail.tsx:122`, `IncidentsPanel.tsx:110`, `useTrail.ts`) | RENDERED |
| P5 | Filter/window state in the query string (L374-376) | `sources`, `days`, `from`, `to`, `gap`, `view`, `exam` from `searchParams` `AppUserDetail.tsx:66-76` | RENDERED |
| P6 | Raw responses, no envelope unwrapping (L360-365) | portal `fetchApi` wraps everything in its own `{success,data}` (`sd()` pattern); the page unwraps the PORTAL envelope, not ours | n/a (our bodies are raw; correct) |
| P7 | "What may be empty": say "this device is on an older build" vs "no activity" (L390-415) | push permission yes (`SnapshotHeader.tsx:498`); timeline empty states are generic ("No activity for these sources in this window", `Timeline.tsx:335,346`) — no appVersion-aware copy | partial |
| P8 | Coverage contract (L417-441) — snapshot/timeline/incidents/note/support-code fields | spot-checked: `permissionGranted`, `deviceCountCapped`, `lastClientEventType`, `limitClamped`, `perSourceRowCap`, `truncatedSources` all referenced in `components/features/app-users/*` | RENDERED (spot-check, not exhaustive) |
| P9 | Answer "which exams has this user bought?" from `entitlements[]` only (L436-437) | `SnapshotHeader.tsx:6,296` ("the ONLY answer…") | RENDERED |
| P10 | Revoke control sends `?exam=` (picker from entitlements, `all` separately confirmed) (L438-439) | lives on the `/app-users` drawer, not the detail page: `AppUsers.tsx:1100-1107,1221-1284` (exam picker + `revokeAllConfirm`) | RENDERED (different location than proposed) |

**Verdict: implemented (~9/10 rendered, 1 partial).** The shipped page follows the proposed IA and adds an exam focus bar and a sessions view. The one partial is the empty-state copy: it never says "this user is on an older build".

---

## WHAT_CHANGED_2026-09-21.md — §4 per-module portal list and §7 gaps, status today

### §4 "What you must change on the portal"

| Module | Portal action required | Status today (evidence) |
|---|---|---|
| Users | exam filter; render `entitlements[]`; revoke `?exam=` | RENDERED — `AppUsers.tsx` `ExamSelector`, entitlements rg hit, revoke with exam + "all" confirm `:1221-1284` |
| Support / activity trail | show exam per timeline row; incidents `?exam=` with exclusion label | RENDERED — T4, T5 |
| Transactions & orders | use `reconcileHint`; `?exam=none`; never grant `UNRESOLVED_EXAM` | RENDERED (spot) — `Transactions.tsx:84,102-118`, `reconcileHint` rg hit. Grant-blocking not verified |
| Offers | exam on create + list; delete `bonusDays` control | RENDERED (spot) — `Offers.tsx:190-193` exam plans; `bonusDays: 0` hard-set `:458` |
| Banners | handle 400; one exam per banner | partial — `Banners.tsx:9,29-30` exam filter; no explicit unknown-exam 400 handling found |
| Analytics | global selector; normalise; "since" label; never compare across deploy | RENDERED on `/engagement`; partial globally (G1/G2) |
| Purchase funnel | new panel | RENDERED as its own page, not a panel (G10) |
| Notifications broadcast/segment | exam selector; show echoed `topic`/`exam` | RENDERED — `AppUsers.tsx:657-668` ("Name the audience the SERVER echoes back") |
| Notification rules / Filter config / Media | nothing | n/a |
| Surveys | `targetExams`; `breakdown=exam` labelled retro-attributed | partial — `targetExams` in `SurveyBuilderModal.tsx`; `breakdown='exam'` + null-bucket caveat `SurveyResultsView.tsx:122-139,240,260`; no "retro-attributed" wording found |
| Feedback | exam filter + "no exam recorded" bucket; keep `examId` ≠ `context.examIds` | RENDERED — `ReportsTab.tsx:57`; `ReportTargetCard.tsx:213` (`contentExamIds`) |
| Exams catalogue | every selector from `GET /sme/exams` | RENDERED (`components/ui/ExamSelector.tsx`) |
| Content ingest | new ingest → `/sme/content/*` | partial — creates/bulk via `/sme/content/*` (`content.api.ts:135`); lists, get, delete still `/cms/*` (AUDIT §1.2) |
| Content docs | tag on create; filter list by exam | partial — `examIds` pushed (`app/services/content_push.py:70,115`); no exam filter on `Documents.tsx`/`Content.tsx` (no `exam` match) |

### §7 "Known gaps that remain"

| # | Gap | Today |
|---|---|---|
| 1 | No per-exam history before 2026-09-21 | Still true (structural). Portal says so (U7–U10). Raw `unknownRows` on `api_usage` should be ~0 now that 30 days have passed 2026-09-21; the rollup's lasts until ~2027-01-19 |
| 2 | Client events carry no exam | Still true: no `exam` on `model AuthEvent` (`schema.prisma`). Portal shows "Directional" `InsightPurchaseFunnel.tsx:172` |
| 3 | `clientErrors` / `authEvents` cannot be exam-scoped | Still true; both labelled (I9, I17) |
| 4 | `/abandoned` TAP signal not exam-filtered | Still true on our side (no change since). Portal copy not checked (offers doc group) |
| 5 | `context.examIds` undefined for simulation questions | Still true; portal tolerates it (`ReportTargetCard.tsx:213` `?? null`) |
| 6 | App does not send `metadata.exam` on client events | Still true (no sink column) |
| 7 | `/cms/*` has no retirement date | **Superseded**: owner ruling 2026-11-03 makes keyed parity and migration a lane (AUDIT §2.2) |
| 8 | "`question-quality` does not read the exam dimension" | **STALE on our side**: it does, since `bfae389` (I2). The live gap is in the portal: proxy `M:5880-5905` drops `exam`, and the page has no selector |

---

## Summary

| Doc | Verdict | Rendered / rows | Missing or wrong (count) | Top consequential |
|---|---|---|---|---|
| SME_USAGE_ANALYTICS.md | implemented | ~21/24 | 3 gaps + 3 stale passages | `isPremium` exam-scoped in code, doc says not (U14); "30d" labels on window-following numbers (U15/U16); startDate/endDate never sent (U3) |
| SME_INSIGHTS_API.md | partial | ~17/19 | 2 gaps + 3 stale passages | question-quality `?exam=` dropped by proxy and no UI (I2); topic/suspectRate/range not exposed (I7); doc contradicts itself on QQ exam |
| SME_METRICS_DECODER.md | partial (≈half the watch-outs stale) | 9/16 still accurate | 6 stale, 2 wrong | 6,000 cap → 40,000 (D1); Apple-events leak fixed (D2); "Actions 30d" window (D5); UI "paid or trial" wording (D6) |
| SME_ANALYTICS_FRONTEND_GUIDE.md | implemented with gaps | ~21/26 | 5 partial | selector per-page, not global or persisted (G1/G2); no purchase-funnel panel or All-exams link (G10); `subject` and route humanising dropped (G21/G23) |
| SME_ACTIVITY_TRAIL_API.md | implemented | 14/14 (1 not verified) | 0 | — |
| SME_USER_DETAIL_PAGE_GUIDE.md | implemented | ~9/10 | 1 partial | no "older build vs nothing happened" empty-state copy (P7) |
| WHAT_CHANGED_2026-09-21 §4/§7 | §4: 9 rendered, 5 partial; §7: 6 still true, 2 superseded or stale | — | 5 partial | content-doc list has no exam filter; banners 400 handling; survey retro label; §7 #8 stale |

**Exam selector, page by page:** `ExamSelector` + `exam` sent on `/engagement`, `/analytics`, release-health, notification-fx, onboarding-funnel and purchase-funnel. **Not** on `/insights/question-quality` or the `/` dashboard.
**`examCoverage` "since" label:** rendered wherever the API returns `examCoverage` (summary, dau, paywall-hits, heatmap, endpoints, release-health). Not rendered anywhere it is absent, which is correct.
**Four reshaping routes:** normalised once in `unwrapExamRows`.
**`conversionPct`:** null shown as "no views — no rate" and as a gap in the line, never 0. Values over 100 are annotated, not clamped.
**Windows:** Engagement offers 7/30/60/90. The two raw-sourced cards say "last 30d only — window not applied". Release health is capped at 30. The trail offers 90 but renders `retention.note`. The real problem is the other direction: "Actions 30d" and "Premium vs free (last 30 days)" follow the 7/60/90 selector while the label still says 30 days.

## Contradictions

1. **`isPremium` on `GET /sme/analytics/users?exam=`.**
   - Doc: SME_USAGE_ANALYTICS.md L159, *"`isPremium` on these rows is still the person-level `premiumExpiresAt > now` mirror — it is **not** re-scoped to the exam."*
   - Code: `sme-analytics.service.ts:302-304`, `isPremium: exam ? !!examPremium : …mirror…`, introduced in `f6329e6`, which WHAT_CHANGED's own commit list names as "per-exam `isPremium`".
   - Portal copy built from the doc is wrong under an exam filter: `EngagementAnalytics.tsx:916`, *"Premium on ANY exam (paid or trial) — not necessarily the exam selected above."*
2. **Question-quality `?exam=`.**
   - These say it is not supported: SME_INSIGHTS_API.md L97-99 ("does **not**; that gap is still open"), L1395 ("still open after `4cfe6c0`"); WHAT_CHANGED §7 #8; README:68 (per AUDIT §2.3).
   - The same doc says it is supported: SME_INSIGHTS_API.md §1 query table, L179.
   - Code: the backend supports it since `bfae389` (`sme-content-quality.dto.ts:92-104`). The portal proxy `M:5880-5905` does not forward it.
3. **"Actions 30d" / "Premium vs free (last 30 days)".**
   - Doc: decoder L186 says "in the last 30 days", and the key name is `actions30d`.
   - Code: both numbers are computed over `since = window(q)` (`service:257,287,412,430`), which follows the portal's 7/60/90 selector (`EngagementAnalytics.tsx:395,379`).
   - UI labels `:544,546,907` say 30 days.
4. **Trial vs `premiumExpiresAt`.**
   - Decoder L188 and the Engagement premium card (`:556-557`, "active trials count as *free*") agree with the code: trials write `trialEndsAt`, not `premiumExpiresAt`.
   - The same page's user-explorer header (`:907`) and crown (`:916`) say "(paid or trial)".
5. **Time-to-read.**
   - SME_USAGE_ANALYTICS.md L248: "derivable only from 2026-07-23 onward".
   - SME_INSIGHTS_API.md L686-706 and the response: `timeToRead.derivable: false`.
6. **`auth_events` consumers.** SME_USAGE_ANALYTICS.md §7 L411-412 says "No SME endpoint reads them yet". Four do: release-health, onboarding-funnel, purchase-funnel and the activity trail.
7. **Error body shape.**
   - SME_INSIGHTS_API.md L84-86: "Nest default `{ statusCode, message }`".
   - SME_ACTIVITY_TRAIL_API.md and AUDIT §2.1: `{success:false,message,…}` from the exception filter.
8. **Per-dispatch taps.**
   - SME_INSIGHTS_API §3 and SME_USAGE_ANALYTICS §4b say "push delivery is not tracked at all" and never mention per-dispatch `tapped`.
   - Code: `notification_dispatch.tapped_at` exists (`schema.prisma:2174`, `3c8bc41`).
   - SME_METRICS_DECODER §5.1 and the portal (`InsightNotificationFx.tsx:141-160`) already document it.
9. **Decoder vs fixed code.**
   - §1 / §8 #2 "6,000 rows": the cap is now 400 pages (`M:5520`).
   - §1 / §8 #1 "Apple events inflated": fixed at `M:5644-5646` and `M:4356-4364`.
   - §8 #4 `[object Object]`: fixed at `InsightOnboardingFunnel.tsx:525-546`.
   - §2 L186 lists the tooltip as still wrong, while its own §8 #7 lists it closed.
10. **Purchase funnel.** FRONTEND_GUIDE L387 ("purchase-funnel panel beneath [premium-engagement + paywall-hits]") and WHAT_CHANGED §4 ("New panel") both call it a panel on the console. It is a standalone page, and Engagement links to it only when an exam is selected (`EngagementAnalytics.tsx:599`).
11. **Version ladder.** "Current store build is 2.0" in SME_USAGE_ANALYTICS L228, SME_INSIGHTS_API L358 and SME_METRICS_DECODER §4. App 2.1 is in the field and 2.2 ships 2026-11-03.
12. **Typo in a shipped caveat.** `sme-purchase-funnel.dto.ts:43`: "mixes a retro-attributed **numerator denominator** with an exactly-attributed numerator". The doc's paraphrase (SME_USAGE_ANALYTICS L324-327) reads correctly; the server string doesn't.
13. **Brief vs code.** The brief lists "`/analytics` and `/` dashboards" among screens with the exam selector story. `/` (Dashboard) has none: all three dashboard components call `getSmeAnalytics()` with no exam.

## Processes stopped

None started. Every command was a one-shot read: `cat`, `sed`, `grep`, `rg`, `awk`, `git log`, `wc`, `ls`. No background jobs, servers or browsers. No pids.
Files written: this file only.
