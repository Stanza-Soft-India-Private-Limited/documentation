# S07: control screens (`/banners`, `/notification-rules`, `/feedback`, `/filter-config`, `/exams`)

Read-only verifier, 2026-11-03. Production portal v2.44.0 (live captures in `~/Desktop/projects/.playwright-mcp/live/`), production backend `upsc:164` (rolled 14:13 IST). Code read: portal `upsc-document-generator` @ `7b00daf`, backend `main` @ `5eee69c`.

Paths: `M:` = `upsc-document-generator/app/main.py`; `UI:` = `upsc-document-generator/admin-ui/src/`; `B:` = `backend/src/`.

**Method.**
- Every form was traced from the UI payload builder, through the proxy allow-list, to our DTO, to the service, to the column.
- DTO verdicts are **proven, not inferred**. The real global `ValidationPipe` config (`B:main.ts:246–254`: whitelist + forbidNonWhitelisted + transform) was run with `ts-node` against the actual DTO classes, using the exact bodies the portal builds. Probe scripts: `scratchpad/s07/settings_probe.ts`, `scratchpad/s07/plan_probe.ts`. These are local, read-only runs, with no HTTP and no DB.
- Production SQL was run read-only through `scripts/staging-db.ts --prod` (SQL files in `scratchpad/s07/q*.sql`). Internal accounts are `@stanzasoft.com/.ai`, `@prepmonkey.*` and `@designasylum.in`.

Brief corrections:
- The `/exams` page is `UI:pages/ExamAdmin.tsx` (route `App.tsx:180`). There is no `pages/Exams.tsx`.
- The feedback config tab is `FeedbackConfigTab.tsx`.

---

## Headline findings (verified)

| # | Finding | Proof | Sev |
|---|---|---|---|
| 1 | **The Guardrails "Save" and the engine kill switch on `/notification-rules` can never succeed.**<br>• GuardrailsPanel seeds `local` from `GET /settings`, which returns `priority: [...]` (`B:notification-engine/services/notification-settings.service.ts:103–107`).<br>• It then posts the whole object: `save.mutate(local)` `UI:…/notification-rules/GuardrailsPanel.tsx:191`, and the kill switch `save.mutate({...local, enabled:false})` `:205`.<br>• The proxy forwards `priority` (`NOTIFICATION_SETTINGS_FIELDS` M:5099).<br>• `UpdateNotificationSettingsDto` has no `priority` (`B:notification-engine/dto/notification-rule.dto.ts:275–299`), so forbidNonWhitelisted rejects it. | Probe: `settings PATCH = GET body → 400 ["property priority should not exist"]`; the kill-switch body gets the same 400; without `priority` → OK.<br>Prod: `app_config.notification_config` is **NULL**, so defaults have never been overwritten. `sme_audit_log` has **0** `NOTIFICATION_SETTINGS_UPDATE` rows ever.<br>The doc teaches the failing body: SME_NOTIFICATION_RULES_API §7 example includes `"priority"` (lines 356–363). | 🔴 |
| 2 | **Any edit to an existing exam plan returns 400 unless the operator changes both "Grants" and the price.**<br>• PlansTab sends only changed keys (`UI:…/exams/PlansTab.tsx:59–64`).<br>• `UpsertExamPlanDto` makes `planType` and `priceInPaise` **required** (`B:modules/sme/dto/sme-exam.dto.ts:293–309`, no `@IsOptional`).<br>• The service's merge branch (`B:modules/sme/services/sme-exam.service.ts:767–792`) is unreachable for a partial body. | Probe: price-only → 400 `planType must be one of…`; strike-only, strike-removal (`null`) and apple-id-only all → 400 on `planType` + `priceInPaise`; a new full plan → OK.<br>Prod: last `EXAM_PLAN_UPSERT` audit row is 2026-09-16; all 4 `exam_plans.updated_at` = 2026-09-16.<br>SME_EXAMS_API §"This upsert MERGES" (lines 310–319) says a strike-only body is valid. | 🔴 |
| 3 | **"Opened" and "Tap rate" on every rule are structurally 0. No app build reports taps.**<br>The UI copy says "Taps are reported only by app 2.0 and later (in both stores since 17 Sep 2026)" (`UI:…/notification-rules/RuleWorkbench.tsx:309`). | Prod: `notification_dispatch` has 196,370 rows, **0** with `tapped_at`, ever.<br>`api_usage` (30 d) has **0** `POST /api/v1/notifications/tapped`, while 46,756 `GET …/unread-count` were recorded on the same controller family.<br>The local app source (`upsc_app` @ `7645b63`, 10-31) has no caller for `/notifications/tapped`. This clone may be stale; it was not fetched.<br>The route exists: `B:notification-engine/notification-preferences.controller.ts:97`. | 🔴 (UNKNOWABLE metric shown as a measurement) |
| 4 | **Chat-feedback table draws every row as thumbs-down** (already live-proven). `m.rating === 'up'` vs the enum `UP` (`UI:…/feedback/ChatFeedbackTab.tsx:188`). | `feedback_chat_rating_proof.txt`. Prod: 99 UP / 7 DOWN. | 🔴 |
| 5 | **Removing an audience filter from an existing rule silently does nothing.**<br>• Clearing "Active within" or "Inactive for" sets `undefined` (`UI:RuleEditor.tsx:677,685`).<br>• Deselecting the last tier or platform also sets `undefined` (`:375`).<br>• `JSON.stringify` drops the key, and the backend shallow-merges `{...before.audience, ...dto.audience}` (`B:notification-rule.service.ts:179`), so the old filter stays.<br>• The toast says "Rule updated". | Code path. The DTO accepts `null` (`@IsOptional`), so sending `null` would clear it.<br>Latent today: all 13 prod rules have `audience = {}`. | 🟠 |
| 6 | **"Dify forward failures 86" with an amber alert reads as an ongoing failure. It is entirely history.** | Prod: 86 rows are `no_api_key`, dated 2026-07-22 → 2026-10-10. **20 of 20** rows since 2026-10-13 forwarded OK. The summary is all-time (`B:sme-feedback.service.ts:211–213`). | 🟠 |
| 7 | **Survey "Reached 2784" is mostly snoozes.** The close dialog says "The N responses already collected" (`UI:SurveysTab.tsx:181`). | Prod, active survey b2d7bebf: 2,785 rows = **428 answered (15.4 %)** + 2,121 snoozed + 236 dismissed. 10 rows are internal; 425 answered by external users. | 🟠 |
| 8 | **The banner preset `…/open/paywall` opens Home.** There is no `paywall` key in the app router (`DeepLinkTarget.kt:463`, aliases `:476–486`). | Our DTO accepts it (probe: `banner create open/paywall → OK`). 0 banners exist in prod, so nothing has shipped with it yet. | 🟠 |
| 9 | **`pyq_open` is mislabelled and cannot be added from the portal.**<br>• The portal labels any cap that has no registry quota "no feature switch (legacy key for older app builds)" (`UI:FeaturesTab.tsx:232`).<br>• APPSC stores `pyq_open {lifetime, 10}` (prod `exams.feature_caps`), so APPSC's Features tab shows the false "legacy" label on its free-question allowance.<br>• UPSC cannot add it: cap rows = effective ∪ stored ∪ registry (`:85–91`). | Live UPSC capture shows 10 cap rows, the legacy one being **`chat`**, which really is legacy. **Correction to AUDIT §5A.1 item 8:** the UPSC tab has no `pyq_open` row. | 🟠 |
| 10 | **"English only today. Other languages are stored but not populated"** (`UI:RuleEditor.tsx:513`) is half true. | Prod after the 14:13 deploy: the backend accepts and merges per-language copy (`B:notification-rule.service.ts:203–250`); the runner picks each recipient's language (`rule-runner.service.ts:882`).<br>But `i18n_languages` holds **only `en`**, and all **16,249** `user_profiles.ui_language` are NULL. So in practice it is English-only until a language is catalogued. The clause "other languages are stored" is false: no rule has a non-`en` key (prod `copy` keys = `en` ×13). | ⚪ |
| 11 | **Correction to AUDIT §5A.1 item 9 ("Deactivate … no guard visible").** | • The APPSC "Deactivate" button opens a `ConfirmModal` with a fallback explanation (`UI:…/exams/ExamList.tsx:65–110`).<br>• upsc-cse renders "Always active: the fallback for every released app build" instead of a button (`:161–164`).<br>• The proxy also refuses (M:7245).<br>• Retire plan, Archive rule and End banner all confirm (`PlansTab.tsx:243`, `NotificationRules.tsx:448`, `Banners.tsx:253`). | — |

---

## Screen 1: `/banners`

Data path:
- UI: `pages/Banners.tsx`, `components/features/banners/{BannerEditor,BannerCard,bannerFormat}.tsx`, `services/banners.api.ts`.
- Proxy: M:7695–7798 (allow-list `BANNER_PATCH_FIELDS` M:7671, `_refuse_banner_status`).
- Backend: `B:modules/sme/controllers/sme-banner.controller.ts`, DTO `B:modules/sme/dto/sme-banner.dto.ts`, service `B:modules/sme/services/sme-banner.service.ts`.
- Table: `exam_banners`.

Live: `GET /sme/exams`, `/sme/banners` (no params), `/sme/offers`, all 200. 0 banners. UPSC "3-day trial"; APPSC "no trial — the Trial tier is empty here".

### A(i) Numbers

| # | Number (UI) | Definition → proxy → UI | Verdict | SQL / result | Sev |
|---|---|---|---|---|---|
| 1 | Pills "All/Draft/Live/Paused/Ended N" (`Banners.tsx:80–84,148`) | Unpaginated list, counted client-side inside the exam filter | TRUE (0/0/0/0/0) | `SELECT status, exam_id, count(*) FROM exam_banners GROUP BY 1,2` → 0 rows | — |
| 2 | "N showing now" globally and per exam (`:86,154,190,199`) | Server-derived `isLiveNow` (`sme-banner.service.ts:328`) | TRUE | same | — |
| 3 | "3-day trial" / "no trial" per group (`:202–206`) | `exams.trial_days` | TRUE: upsc-cse 3, appsc-group-1 0 | q7 | — |

### A(ii) New/Edit banner modal, field by field (`BannerEditor.tsx`)

| UI field | Payload key (create `:259–271`; patch is diff-only `:131–147`) | Proxy | Our DTO (`sme-banner.dto.ts`) | Column | Notes |
|---|---|---|---|---|---|
| Exam * (active exams) `:353` | `examId` | ✓ | string ≤64, required on create; `assertExists` (`service:68,122`) | `exam_id` | Re-targetable on edit (DTO `:197`) |
| Internal name * (≤120) `:365` | `name` | ✓ | ≤120 | `name` | match |
| Starts/Ends at (IST) `:369,372` | `startsAt`, `endsAt` (`+05:30` appended, `bannerFormat.ts:18–20`) | ✓ | `@IsISO8601`; service `endsAt > startsAt` (`service:270–277`) | `starts_at`, `ends_at` | IST correct |
| Artwork URL + Upload `:396–407` | `imageUrl` (upload via `/sme/media/upload-url`, folder `banners`) | ✓ | https regex, ≤500 | `image_url` | Dimension check is client-only (by design) |
| Destination URL + presets `:441–452` | `destinationUrl` | ✓ | regex `^https://(app\|go)\.prepmonkey\.com/open/` (`banner.constants.ts:52`) | `destination_url` | 🟠 `/open/paywall` preset → app Home (finding 8). 🟡 the `/open/offer/` preset saves with **no code** (probe OK) → app opens the normal paywall (`DeepLinkTarget.kt:264`): harmless, but the "CODE" placeholder is not enforced. Should be `/open/premium` |
| Tier chips Paid/Trial/Lapsed/Free `:462` | `audienceTiers` | ✓ | `@ArrayNotEmpty @IsIn(free,trial,paid,lapsed)` | `audience_tiers` | Default all four = DTO default. Per-exam tier (`banner.constants.ts:12–24`) |
| Platform Android/iOS/Web `:492` | `platforms` | ✓ | `@ArrayNotEmpty @IsIn` | `platforms` | Web is real (`prepmonkey-web/src/lib/home.ts` fetches banners) |
| Min/Max account age `:502,506` | `min/maxAccountAgeDays` (`''` → `null` clears) | ✓ (key membership) | int 0–3650, nullable; service checks min ≤ max on the merged pair | `min/max_account_age_days` | UI has no 3650 cap (server 400 surfaced) |
| Priority (−1000…1000) `:520` | `priority` | ✓ | int −1000…1000 | `priority` | match |
| — | `status` | refused at the proxy (M:7679) and service | declared only to 400 with a hint | — | correct |

- **Fields we accept that the form cannot set:** none.
- **No language dimension at all:** the copy is baked into the artwork, so a Hindi user sees the English artwork. Per-language artwork needs new persistence (flag only).
- **Mutations without confirmation:** Activate, Pause and Resume act immediately with a toast (`BannerCard.tsx:40–76`). They are reversible, which is intended. End confirms (`Banners.tsx:253–275`).

### A.2 Exam scoping

A banner belongs to exactly one exam (DTO required plus FK). The page fetches every banner and filters by exam client-side (`Banners.tsx:38–40,79`). That is correct for an unpaginated list, but the strict upstream `?exam=` is never exercised (C2). The ExamSelector is the standard control.

---

## Screen 2: `/notification-rules`

Data path:
- UI: `pages/NotificationRules.tsx`, `components/features/notification-rules/{RuleEditor,RuleWorkbench,GuardrailsPanel,shared}.tsx`; API `UI:services/api.ts:2280–2595`.
- Proxy: M:5075–5383.
- Backend: `B:notification-engine/notification-rule.controller.ts`, DTOs `dto/notification-rule.dto.ts`, `services/notification-rule.service.ts`, `notification-settings.service.ts`.
- Tables: `notification_rule`, `notification_dispatch`, `notification_rule_run`, `app_config.notification_config`.

Live: `GET /settings`, `/triggers`, `/sme/notification-rules`, all 200. 13 rules: 11 "Live · 100%", 2 Off, "Enabled · 0%" 0.

### A(i) Numbers

| # | Number (UI) | Definition → UI | Window/TZ | Excl. | Verdict | SQL / result | Sev |
|---|---|---|---|---|---|---|---|
| 1 | "All 13 / Live 11 / Enabled·0% 0 / Off 2" | Portal `ruleState()` from `isEnabled`/`rolloutPercent` (`shared.tsx:16–19`). It ignores server `isLiveNow`, which is logically identical (`service:534`); neither reflects the global kill switch, despite the comment at `api.ts:2357` | now | n/a | TRUE | q2: 11 rules enabled at 100 %, `custom`/`pib_today` off at 0 %, 0 archived | — |
| 2 | "Live · 100%" on `reels_today` | Enabled. It does not mean "sending" | — | — | MISLABELLED for an outsider | q2: reels_today **0 sends ever** ("silent if nothing dropped") | 🟡 |
| 3 | Panel "Sent · last 30d" (`RuleWorkbench.tsx:290`) | `count(notification_dispatch)` where `rule_id`, `!is_test`, `sent_at ≥ now−30d` (`service:421–439`). One row per user per dedupe key handed to FCM/feed; not delivery | rolling 30 d, UTC instant (fine) | **no internal exclusion** | TRUE as "handed off". Internal share tiny: pyq_reminder 70,389 vs 70,214 external; continue_learning 10,283 vs 10,186 | q2 (full table in E) | ⚪ |
| 4 | "Opened" / "Tap rate" (`:294–301`) | `count(tapped_at)` | 30 d | — | **UNKNOWABLE.** 0 taps ever recorded; no client reports taps (finding 3). The "not yet available" text is accidentally right; the 2.0 caveat copy is false | q3: 196,370 rows, 0 tapped | 🔴 |
| 5 | Backend `tapRate` | `sent ? … : 0` (`service:438`) returns **0, not null**, when nothing was sent | — | — | Bug class 10 on our side; the portal type says null (`api.ts:2465`) | — | ⚪ |
| 6 | Guardrails "22:00–08:00, max 2/day, Engine enabled" | `GET /settings` = defaults (stored blob NULL) | — | — | TRUE (they are the defaults), but **unchangeable from the portal** (finding 1) | `SELECT notification_config FROM app_config` → NULL | 🔴 |
| 7 | Preview funnel (matched → afterPrefs → afterCaps; `RuleWorkbench.tsx:95–145`) | Server dry run; null ≠ 0 handled | now | — | TRUE (C2 rows 14, 11) | not run (would call preview, read-only but heavy) | — |

### A(ii) Rule editor (`RuleEditor.tsx`, body built at `:317–343`)

| UI field | Payload | Proxy (`_rule_body` M:5103) | Our DTO | Column | Issue |
|---|---|---|---|---|---|
| Trigger (create only) | `triggerKey` | POST only; PATCH refuses (M:5235) | ≤64, must be in the registry | `trigger_key` | ok, immutable |
| Name | `name` | ✓ | ≤120 | `name` | ok |
| Config inputs generated from `configSchema` (`:480–500`) | `config` | ✓ | object; service rejects unknown keys and clamps | `config` | ok |
| Title / body with `{{vars}}` chips, counters 120/500 | `copy: {en:{title,body}}` (`withRuleCopy`, `api.ts:2385`) | ✓ | `@IsLanguageMapOf(RuleCopyDto)`; merged per language (`service:203–250`) | `copy` | English only; `hi`/other cannot be authored or seen (copy at :513, finding 10) |
| Destination select + Entity id (`:520–537`) | `deepLink {type,id}` (id `''` → `null`) | ✓; params capped (M:5110–5118) | `RuleDeepLinkDto` | `deep_link` (shallow-merged `service:176`) | ok. Switching type keeps the old `id` in the draft unless cleared |
| Send time (IST) | `sendAtIst` (omitted for EVENT) | ✓ | `HH:MM`; 400 on EVENT | `send_at_ist` | ok |
| Cooldown days | `cooldownDays` (`''` → **null**) `:335` | ✓ | int 0–365, `@IsOptional` lets null through | `cooldown_days` **NOT NULL** (schema) | 🟡 blanking it on edit → `data.cooldownDays = null` (`service:148`) → Prisma error → 500. Create: `?? 1` is fine |
| Priority | `priority` (`''` → null) `:336` | ✓ | int **0**–1000 | `priority` NOT NULL | 🟡 blank on edit → 500; negative → 400 (probe: `priority must not be less than 0`; UI has no min). Hint "Seeded PAYMENT 10 · STREAK 20 · everything else 50" matches prod (q2), but a new rule defaults to **100** (`service:121`) |
| Tier chips premium/trial/free, platforms | `audience.tiers/platforms` | ✓ | `@IsIn` | `audience` | 🟠 deselecting all = `undefined` = no change (finding 5). Tier is **person-level**, not per exam (`notification-audience.util.ts:128–141`), unlike banners |
| Active within / Inactive for (days) `:672–686` | `audience.activeWithinDays/inactiveForDays` | ✓ | int **1**–365 | `audience` | 🟠 clearing = no change; 🟡 UI `min=0`, so 0 → 400 (probe) |
| Exam scope (All = `null`) `:705–717` | `examId` | ✓ (null survives) | string ≤64; **no existence check** (create `:107`, update `:146`) | `exam_id` (not an FK by design) | ok via dropdown |
| — | `channel` | not in the allow-list | accepted by the DTO | `channel` | cannot be set from the portal (FEED_ONLY unreachable) |

Workbench actions:
- Preview, Test (≤10 ids, proxy M:5302).
- Enable at percent (confirm `RuleWorkbench.tsx:643`) and Turn off (confirm `:658`).
- Archive (confirm `NotificationRules.tsx:448`). There is no un-archive anywhere.

Guardrails panel: every field fails (finding 1). The input `max=5` mirrors the DTO `@Max(5)` (a 400, not a clamp, as C2 found).

### A.2 Exam scoping

- `examId` null = every exam. UPSC-scoped also matches users whose active exam is NULL (warning `RuleEditor.tsx:730`).
- The list's exam and trigger filters are client-side.
- Per-rule analytics are not per exam.
- All 13 prod rules are unscoped (`exam_id` NULL).

---

## Screen 3: `/feedback` (Reports · Chat feedback · Surveys · Config)

Data path:
- UI: `pages/Feedback.tsx` → `components/features/feedback/{ReportsTab,ReportTargetCard,ChatFeedbackTab,SurveysTab,SurveyBuilderModal,SurveyResultsView,FeedbackConfigTab}.tsx`; API `UI:services/api.ts:2958–3285`.
- Proxy: M:6557–6586, 6811–6884, 6889–6995.
- Backend: `B:modules/sme/{controllers,services}/sme-feedback.*`, `sme-survey.*`, DTOs `sme-feedback.dto.ts`, `sme-survey.dto.ts`; config `B:modules/feedback/services/feedback-config.service.ts`.

### A(i) Numbers

| # | Number (UI) | Definition | Window | Excl. | Verdict | SQL / result | Sev |
|---|---|---|---|---|---|---|---|
| 1 | Reports "Showing 1–20 of 64" | `count(feedback_reports)` with filters (`sme-feedback.service.ts:73–102`) | all time | none | TRUE (64); 3 internal | q5: ISSUE 55 (OPEN 1, IN_REVIEW 22, RESOLVED 3, CLOSED 29), FEATURE 9; `exam_id` NULL 38, upsc 24, appsc 2 | ⚪ |
| 2 | Chat "Total feedback 106" | `count(*)` all time | all time | none | TRUE; 8 internal rows, 79 external users | q4 | ⚪ |
| 3 | "Thumbs up 99 / down 7" | enum counts | all time | none | TRUE (cards); the table below draws all as down (finding 4) | q4 | 🔴 (table) |
| 4 | "Dify forward failures 86" + amber alert | `!dify_forwarded AND dify_error NOT NULL`, all time (`:211–213`) | all time | — | MISLABELLED: all of it is 07-22 → 10-10; 0 since the key landed (finding 6) | q4 | 🟠 |
| 5 | "By mode SME 33 / MENTOR 73" | groupBy mode; renders the raw enum uppercase | all time | — | TRUE (70+3, 29+4) | q4 | ⚪ |
| 6 | Top chips (Incomplete 2, Inaccurate 2…) | Last 2,000 rows' chips (`:214–231`) | last 2,000 rows | — | TRUE (all 106 included) | — | — |
| 7 | Surveys "Reached 2784" | `_count.responses`, every status (`sme-survey.service.ts:254`) | all time | none | TRUE as labelled; the tooltip `SurveysTab.tsx:105` is honest; the close dialog `:181` over-states | q6: 428 answered / 2,121 snoozed / 236 dismissed | 🟠 |

### A(ii) Forms

**Report detail** (`ReportsTab.tsx:166–190`):
- Status: `{status, actorNote}` → proxy allow-list (M:6826) → `UpdateReportStatusDto` (enum + optional note) → `feedback_reports.status`. PLANNED/SHIPPED on an ISSUE → `BizException`.
- Reply: `{text, actorNote}` (M:6837) → `CreateReportReplyDto` → `feedback_report_replies`.
- Both send the user an **English-only** push ("Update on your feedback", "You have a reply", `sme-feedback.service.ts:170–198`), with no confirmation. The toast states that a push is sent.

**Survey builder** (`SurveyBuilderModal.tsx:264–295`):

| UI field | Create | Patch | Proxy allow-list (M:6902, 6926) | DTO | Note |
|---|---|---|---|---|---|
| Title / description | string | string; description `''` → `undefined` (dropped) | ✓ | `@IsStringOrLanguageMap` | 🟡 a description cannot be cleared on edit (merge keeps the old one) |
| Questions | array, `id` kept | only while DRAFT | ✓ | `@IsArrayOrLanguageMap` | ok; 409 when ACTIVE |
| Target tiers / platforms / aspirant types / exams | arrays (`[]` = everyone) | same | ✓ | `IsString each` (no enum check) | ok; `targetExams` correct |
| Priority | int | int | ✓ | `@IsInt` | ok |
| Start at (create only) | ISO from browser local TZ (`:37–42`) | not sent (correct: `UpdateSurveyDto` has no `startAt`) | ✓ | `@IsString` | 🟡 no "IST" label; local-TZ dependent |
| End at | ISO | ISO; `''` → `undefined` | ✓ | `@IsString` | 🟡 an end date cannot be removed on edit |
| — | `…ByLanguage` / language maps | — | the keys would pass | accepted | **No UI** (C2 row 12). Prod: `surveys.i18n` NULL ×3 |

Survey actions:
- Activate and Close both have a ConfirmModal (`SurveysTab.tsx:174–195`).
- Notify `{title, body}` sends a push to the cohort, with English-only copy.

**Config tab** (`FeedbackConfigTab.tsx:146–153`): sends only the dirty sections `{chipSets|survey|limits}` (proxy M:6883). Our side:
- `chipSets` is validated: known set keys, key regex, unique keys, non-empty label (`feedback-config.service.ts:99–130`).
- 🟡 `survey.*` and `limits.*` numbers are **not validated at all**. The DTO only has `@IsObject`, and the service spreads them in (`:160–170`). Probe: `snoozeDays: -5` → OK. A 0 or negative `reportsPerDay` would reach the app unchecked.
- Chip `label` is one string, so there is no per-language label. That needs new persistence (flag only).

### A.2 Exam scoping

- Reports: `?exam=` is forward-only. 38 of 64 rows are NULL, which the "No exam recorded" option handles.
- Chat feedback: **no exam filter** on summary or messages, although `chat_message_feedback.exam_id` exists (87 of 106 NULL). `examId` is not rendered (C2).
- Surveys: `targetExams` + `breakdown=exam`.
- Config: global (same chips for every exam).

---

## Screen 4: `/filter-config`

Data path:
- UI: `pages/FilterConfig.tsx`; API `UI:services/api.ts:2913–2950`.
- Proxy: M:6483–6549 (`_filter_config_params`); shared-sample M:6514 → keyless `/cms/mains`.
- Backend: `B:modules/sme/controllers/sme-filter-config.controller.ts`, DTO `sme-filter-config.dto.ts`, service `sme-filter-config.service.ts`.
- Storage: `app_config.filter_config.byExam`.

### A(i) Numbers

| # | Number | Definition | Verdict | SQL | Sev |
|---|---|---|---|---|---|
| 1 | Per-subject question count (Art & Culture 78, CSAT 1040, Current Affairs 24) (`FilterConfig.tsx:538`) | Prelims: active, in exam or `*`, playable (`boardDeleted=false AND correctAnswer NOT NULL`) (`service:133–156`); all languages | TRUE (78/1040/24 reconcile; 1 language in prod). Unlabelled as "playable" (C3) | `SELECT subject, count(*) FILTER (WHERE "boardDeleted"=false AND "correctAnswer" IS NOT NULL) FROM pyq_papers WHERE "isActive" AND "examIds" && ARRAY['upsc-cse','*'] …` → 78/1040/24 | ⚪ |
| 2 | Flip modal "This bulk-mutates all N question row(s)" (`:293`) | N = mains **active** rows | MISLABELLED: the flip updates `{subject, examIds}` **including inactive rows** (`service:294–302`), so N understates | not run (would need mains inactive count) | 🟡 |
| 3 | Shared-sample `shared/sampled/total/complete` | Portal-built from the first 100 `/cms/mains` rows | `complete = len(page) >= total` is a correct check; `shared` is a ≤100-row sample | — | ⚪ |

### A(ii) Edit-override modal

| UI field | Payload (three-state unset/clear/set, `:60–112`) | Proxy (M:6505) | DTO | Note |
|---|---|---|---|---|
| Display name | `displayName` string, or `null` | ✓ | `@IsStringOrLanguageMap` | A string = English only; per-language maps are not authorable; a Hindi-only override renders "—" (C3) |
| Icon (from `validIconKeys`) | `iconKey` | ✓ | string; service checks the list | ok |
| Visible / Hidden | `hidden` | ✓ | bool/null | ok |
| Sort order | `sortOrder` (`''` → null) | ✓ | int/null | ok |

Optional/GS flip: `{isOptional}` (M:6546) → `SetMainsOptionalDto`, with confirm. Prod overrides: exactly one (APPSC prelims "Mental Ability" iconKey); 1 `FILTER_CONFIG_UPDATE` audit row (09-23).

### A.2 Exam scoping

`?exam=` is passed through on all three routes. An unknown slug → 400, surfaced. The Prelims/Mains toggle sits beside the exam combobox.

---

## Screen 5: `/exams`

Data path:
- UI: `pages/ExamAdmin.tsx`, `components/features/exams/{ExamList,SettingsTab,FeaturesTab,PlansTab,ContentCountsTab,CreateExamModal}.tsx`, `services/exams.api.ts`.
- Proxy: M:7084–7316 (allow-lists M:7116–7126, `_validate_exam_config`).
- Backend: `B:modules/sme/controllers/sme-exam.controller.ts`, DTO `sme-exam.dto.ts`, service `sme-exam.service.ts`.
- Tables: `exams`, `exam_plans`.

### A(i) Numbers

| # | Number | Verdict | SQL / result |
|---|---|---|---|
| 1 | "upsc-cse · 3-day trial · exam 23 May 2027" / "appsc-group-1 · no trial · exam 24 Jan 2027" | TRUE | q7: trial_days 3 / 0; exam_date 2027-05-23 / 2027-01-24; policy history 14 → 3 on 2026-09-25 |
| 2 | Plans "₹5,900/yr (strike ₹7,900)", "₹599/mo (strike ₹799)" | TRUE | `exam_plans` upsc: 590000/790000, 59900/79900; APPSC 499900/699900, 49900/69900 |
| 3 | Effective caps "3/day ×8, content_doc 1 per subject, pyq_variation Premium only" | TRUE (global `app_config.feature_caps`; UPSC has no overrides) | q7 |
| 4 | Content counts tab | not exercised live; C2 row 3 | — |

### A(ii) Field by field (SME_EXAMS_API §7–§12, as the owner asked)

| Doc § | UI field | Payload | Proxy | DTO | Verdict |
|---|---|---|---|---|---|
| §1 create | id / names / exam date / sort (`CreateExamModal.tsx:30–36`), `isActive:false` | POST | slug regex, shortName ≤24, default inactive (M:7186–7209) | `CreateExamDto` | ok |
| §2 identity | Display/short name, exam date, sort (`SettingsTab.tsx:79–82`) | diff only | ✓ | `UpdateExamDto` | ok |
| §5 activation | Activate/Deactivate (`ExamList.tsx:45`) with confirm + content-count check | `{isActive}` | upsc-cse guard M:7245 | `@IsBoolean` | ok |
| — access tier | radio free/paid; paid blocked without an active plan (`SettingsTab.tsx:96`); confirm (`:95`) | `accessTier` | ✓ | `@IsIn(free,paid)` | ok |
| — Apple group | `appleSubscriptionGroupId` (`''` → null) | ✓ | ✓ | `@IsString` ≤32, null passes | ok |
| §7 flags | toggles from `/feature-registry` (`FeaturesTab.tsx:73–80`) | `featureFlags` partial | bool check M:7140 | `@IsObject`; service validates keys | ok |
| §8 caps | cap editor; type-switch confirm (`:289–301`) | `featureCaps` partial, `null` = global | type/limit check M:7148 | `@IsObject`; closed key list | 🟠 `pyq_open` not addable, false label (finding 9). Doc §8 key list omits it |
| §9 trial | Trial days 0–90 with confirm (`SettingsTab.tsx:221, 343–348`) | `trialDays` | 0–90 M:7133 | `@Min(0) @Max(90)` | ok; works in prod (09-25 change) |
| §10 logo | upload `exam-logos` / Remove (`:160–168`) | `logoUrl` / null | ✓ | ≤500, null | ok |
| §11 paywall copy | titles, highlight, benefits, rows (`:88–91`) | `paywallContent` (English object) | ✓ | `@IsObjectOrLanguageMap` | per-language map not authorable (C2 row 21). UPSC `paywall_content` NULL (legacy table); APPSC has 9 English keys |
| plans | Plan, Grants, Price, Strike, Apple id, Razorpay id (`PlansTab.tsx:59–64`) | changed keys only | price/strike checks M:7286–7301 | **`planType` + `priceInPaise` required** | 🔴 finding 2. Also 🟡 a store id cannot be cleared (blank is not sent, `:63–64`; a warning shows instead) |
| plans retire | Retire with confirm (`:230, 243`) | DELETE | ✓ | — | ok |

### A.2 Exam scoping

Per exam by construction. The page keeps its own exam list (`exams.api.ts:140–144`), not the shared ExamSelector. That fits an admin screen.

---

## B. MEANING

Owner rulings applied:
- "The numbers are there BUT those numbers are not explaining anything … explain in simple terms … the portal might already do some things well and corrections may not be needed at all."
- Template: `/insights/purchase-funnel`. That means a one-line subtitle, a scope line ("for all exams / last 30 days"), plain counts, labelled losses, one ratio with its caveat, and the standard exam selector + window pills.

These are control screens, so most panels are forms and pass by being exact. The proposals below are one sentence per panel, with the ExamSelector as the scoping control.

| Screen / panel | One plain sentence | Bucket | Derivation / persistence |
|---|---|---|---|
| Banners header + groups | "UPSC: 0 banners showing now, carousel empty for everyone; APPSC: same, and APPSC has no trial so 'Trial' reaches nobody." | **KEEP.** The page already speaks this way (`Banners.tsx:215–220`). Only change: ExamSelector as the standard scope with a scope line | Computable today |
| Banner modal | — | **KEEP.** Fix the `/open/paywall` preset → `/open/premium` | — |
| Rules list | Today "Live · 100%". Proposed: "Live for everyone · sent to 70,389 people in 30 days · opens not measured" per row; reels_today: "Live · sent 0 in 30 days (nothing was published)". | **COMPACT.** Merge the status badge with a 30-day sent count; drop "Live" as a success signal | `notification_dispatch` `count(*) WHERE rule_id, !is_test, sent_at ≥ now−30d`, permanent table, computable today. Opens need app-side tap reporting (**new capture**: no client calls `/notifications/tapped`) |
| Rule panel "Sent / Opened / Tap rate" | "Sent to 2,289 people in 30 days. Whether anyone opened them is not measured yet." | **COMPACT.** Replace "Opened 0 / not yet available" and the false 2.0 caveat with one honest line until taps exist | Same |
| Rule panel runs | "Last ran 18:00 IST, matched N, sent M; K skipped (cap/quiet hours/opted out)." | **KEEP** (the funnel already labels losses; matches the template) | `notification_rule_run` |
| Guardrails | "Engine on; quiet 22:00–08:00 IST; at most 2 reminders a day per person." | **KEEP** the text; the save is broken (finding 1) | — |
| Rule editor | — | **KEEP**; fix the clear/null bugs; replace "English only today…" with "Only English is set up today; other languages appear here once added" | — |
| Reports tab | "64 reports, 1 open, 25 in review (oldest 12 days); median first reply 2.6 h." | **COMPACT.** Add one status-sentence tile above the table; the table stays | `feedback_reports` + `feedback_report_replies` (q5), computable today |
| Chat feedback cards | "106 ratings from 79 people: 99 up, 7 down (93 % positive). Since the 13 Oct fix every rating reaches Dify." | **COMPACT.** Merge four tiles into one sentence; turn "86 failures" into "0 failures since 13 Oct (86 before the key was set)"; add the exam selector (column exists) | `chat_message_feedback` (`rating`, `dify_*`, `exam_id`), computable today |
| Chat table | — | **KEEP** after fixing the thumb bug; add `examId` | — |
| Surveys list | "Help us make PrepMonkey better: shown to 2,785 people, 428 answered (15 %), 2,121 said later, 236 dismissed." | **COMPACT.** Replace "Reached 2784" with answered / later / dismissed; fix the close-dialog wording | `survey_responses.status`, computable today |
| Survey results | — | **KEEP** (C2: NPS gating, exam caveat already honest) | — |
| Feedback config | — | **KEEP**; add server-side bounds | — |
| Filter-config table | "Art & Culture: 78 playable questions, shown as 'Art & Culture' with the art icon, visible." | **KEEP.** Label the count "playable questions" | Computable today |
| Exams list chip | "UPSC: paid, 3-day trial for new signups, exam 23 May 2027." | **KEEP** (already plain) | — |
| Exams Features | — | **KEEP**; relabel `pyq_open` "Free questions per account (lifetime)" and make it addable | Registry change on our side |
| Plans | "Annual ₹5,900 (shown struck from ₹7,900), monthly ₹599 (from ₹799)." | **KEEP**; fix the save path | — |

Shareholder questions:

- **(5) notification outcomes:** *partially.* Sent per rule is answerable and persisted; opens are **not**. Tap persistence exists (`notification_dispatch.tapped_at`), but nothing writes it; that is an app-side gap, flag only.
- **(4) question quality:** *partially*, via reports linked to content (`context_type/context_id`) and chat down-votes.
- (1), (2), (3) and (6) are not this screen group's job.

What the multi-exam + multi-language model needs each form to carry (minimal change, no design):

| Form | Exam | Language |
|---|---|---|
| Banner | ✓ already | Needs per-language artwork (**new persistence**) or a language audience chip (**new column**) |
| Rule | ✓ (`examId`) | `copy.{code}` already persisted; the form needs a language tab fed by `/sme/i18n/languages` (backend ready; prod catalogue `en` only). Rule tier is person-level, not per exam: per-exam tier targeting needs backend work (documented gap) |
| Survey | ✓ (`targetExams`) | `…ByLanguage` already persisted; form tab needed |
| Feedback config | none (global) | Chip labels need per-language (**new persistence**). Report/reply push copy is hard-coded English on our side |
| Filter override | ✓ (`?exam=`) | `displayName` map already persisted; form needs a language tab |
| Exam paywall | ✓ | `paywallContentByLanguage` already persisted; form tab needed |
| All lists | — | `X-Lang` cannot pass through the proxy (fixed headers M:4369) |

---

## C. GAP (docs → proxy → UI)

| Item | State |
|---|---|
| `GET /sme/banners?exam=` strict filter; `GET /sme/banners/:id` | proxied-only (C2) |
| Rule `channel` | backend-only (not in `RULE_WRITE_FIELDS` M:5095) |
| Rule `copy.{lang}`; survey `…ByLanguage`; filter `displayNameByLanguage`; exam `paywallContentByLanguage` | proxied-only, no UI |
| `GET /sme/notification-rules/:id`; `GET /sme/surveys/:id/responses` | proxied-only (UI fns unused) |
| Rule `isLiveNow` | proxied-only (re-derived) |
| Chat feedback `examId`, `updatedAt`; any exam filter on chat | backend-only / not rendered |
| `pyq_open` cap | absent from docs, partially rendered with a false label |
| `/triggers` `sampleVars` | doc promises; backend does not return it; UI falls back (C2) |
| `/sme/i18n/*` (language catalogue for any language tab) | absent from the portal |

### C.1 Window controls

Owner ruling: every data screen gets Exam selector + window pills + custom from–to + Refresh. The table below covers each windowed route on these screens.

| Route (screen) | Window params our backend accepts | DTO / controller cap | Proxy forwards | UI today | Source retention | Needed for pills + custom range |
|---|---|---|---|---|---|---|
| `GET /sme/notification-rules/:id/analytics` (rule panel) | `days` only (`notification-rule.controller.ts:162–168`, `ParseIntPipe`, **no max**); **no** `from`/`to`/`startDate`/`endDate` | none on our side (`days` unbounded); rolling `now − days`, a UTC instant | `days`, 1–365 (M:5368–5383) | fixed `days=30` (`api.ts:2580`, `RuleWorkbench.tsx:277`), no pills | `notification_dispatch` is permanent (no purge in `notification-engine/services`) | **New backend params** `from`/`to` (IST dates) + exam. Pills map to `days` today |
| `GET /sme/notification-rules/:id/runs` | `limit` only | `ParseIntPipe`, no max on our side | `limit` 1–100 (M:5354–5363) | `limit=20` | `notification_rule_run` permanent | New `from`/`to` needed |
| `GET /sme/notification-rules` (list; proposed 30-day sent column) | none | — | — | — | — | New derived field or endpoint needed |
| `GET /sme/feedback/reports` | **`from`/`to`** as IST `YYYY-MM-DD`, inclusive (`sme-feedback.dto.ts:74–82`, service `istRange` `sme-feedback.service.ts:309–320`) | `@IsString`, format not validated; no span cap | ✓ `from`, `to`, `exam` (M:6557–6586) | From (IST) / To (IST) inputs already present; no pills | permanent | Pills can map to `from`/`to` with **no backend change** |
| `GET /sme/feedback/chat/summary` | none (all-time) | — | none (M:6844) | none | permanent | New `from`/`to` + `exam` (column exists) |
| `GET /sme/feedback/chat/messages` | none (`rating`, `mode`, `page`, `limit`) | `limit` ≤100 | the same four (M:6852–6861) | none | permanent | New `from`/`to` + `exam` |
| `GET /sme/surveys`, `/:id`, `/:id/results` | none (`breakdown` only) | — | `breakdown` (M:6966) | none | permanent (`survey_responses`) | New `from`/`to` on results/counts if wanted (`answered_at` / `created_at` exist) |
| `/sme/banners`, `/sme/filter-config/*`, `/sme/exams*` | not windowed (config) | — | — | — | — | Exam selector + Refresh only; no window applies |

`api_usage`'s 30-day retention does not touch any route here. All sources are permanent domain tables, so any window is fully backed.

---

## D. Contradictions

1. **SME_NOTIFICATION_RULES_API §7 PATCH /settings example** sends `priority` → 400 (DTO). The same doc says `maxPerUserPerDay` is "corrected", while the DTO 400s it (C2). Our controller's own Swagger text says "corrected, not rejected" (`notification-rule.controller.ts:66`): also false.
2. **SME_EXAMS_API plans §"MERGES"**: "`{strikePriceInPaise}` on its own is still checked against the stored price" vs the DTO requiring `planType` + `priceInPaise` (probe 400). Our own service comment (`sme-exam.service.ts:767–772`) says "the portal's price editor sends only a price"; that request is now rejected one layer earlier.
3. **Portal copy "Taps are reported only by app 2.0 and later (in both stores since 17 Sep 2026)"** (`RuleWorkbench.tsx:309`, echoed in proxy docstring M:5373) vs prod: 0 taps, 0 `/notifications/tapped` calls.
4. **Portal copy "English only today. Other languages are stored but not populated"** (`RuleEditor.tsx:513`); also `api.ts:2326` and schema comment `notification_rule.copy` ("Only `en` is populated today"). Per-language copy is live on our side. Prod has only `en` catalogued, so "stored" is false and "English only" is true in practice.
5. **AUDIT §5A.1 item 8** ("Features tab lists 10 cap keys + the `pyq_open` legacy row"): the UPSC tab's legacy row is `chat`; `pyq_open` shows only on APPSC (which stores it).
6. **AUDIT §5A.1 item 9** ("Deactivate … no guard visible"): there is a ConfirmModal, and upsc-cse cannot be deactivated (UI + proxy).
7. **Brief file names**: `pages/Exams.tsx` → `pages/ExamAdmin.tsx`; "ConfigTab" → `FeedbackConfigTab.tsx`.
8. **Rule editor hint "Seeded … everything else 50"** is true for the seeded rules, but a new rule gets priority **100** (`notification-rule.service.ts:121`; schema default 100).
9. **Portal comment** that `isLiveNow` = "not globally frozen" (`api.ts:2357`): our `isLiveNow` ignores the kill switch and shadow mode (`service:534`).
10. **Portal `tapRate` typed nullable** (`api.ts:2465`): ours returns 0 when sent = 0 (`service:438`).
11. Carried from C2/C3, unchanged: `/open/paywall` preset; `responseCount` ≠ answers; SME_EXAMS_API §2/§9/§12 "14-day" stale (prod 3); §5 "broadcast cannot be exam-scoped" false; PAGE_GUIDE `sampleVars` and tapped-route errors.

---

## E. SQL run (production, read-only) and processes

All via `npx ts-node -T scripts/staging-db.ts --prod`, output filtered through `rg -v 'eyJ|rzp_|sk-'`. Files: `scratchpad/s07/q1.sql`–`q7.sql`. `information_schema.columns` was checked first.

- **q1** `exam_banners` by status/exam → **0 rows**. `sme_audit_log` actions:

  | Action | Count | Dates |
  |---|---|---|
  | EXAM_PLAN_UPSERT | 12 | 09-08 → 09-16 |
  | EXAM_UPDATE | 12 | → 10-14 |
  | FILTER_CONFIG_UPDATE | 1 | — |
  | NOTIFICATION_RULE_ENABLE | 13 | 08-28 |
  | NOTIFICATION_RULE_UPDATE | 2 | 10-01 |
  | FEEDBACK_REPORT_STATUS | 92 | — |
  | FEEDBACK_REPORT_REPLY | 65 | — |
  | SURVEY_* | 6 | — |
  | **NOTIFICATION_SETTINGS_UPDATE** | **0** | — |
  | BANNER_* | 0 | — |

- **q2** per rule, 30 d non-test dispatch (all / external):

  | Rule | Sent all | Sent external |
  |---|---|---|
  | pyq_reminder | 70,389 | 70,214 |
  | continue_learning | 10,283 | 10,186 |
  | trial_ending | 2,289 | 2,288 |
  | weekly_progress | 2,189 | 2,173 |
  | trial_started | 2,140 | 2,140 |
  | streak_unfilled | 550 | 541 |
  | streak_completed | 60 | 55 |
  | payment_success | 10 | 10 |
  | subscription_ending | 5 | 5 |
  | subscription_ended | 3 | 3 |
  | reels_today, custom, pib_today | 0 | 0 |

  Taps 0 everywhere. `copy` keys `en` only. Every rule has `exam_id` NULL. `notification_config` NULL. 0 archived.
- **q3** `notification_dispatch`: 196,370 rows, 0 tapped, 64 tests. `api_usage` 30 d: no `/notifications/tapped` route at all.
- **q4** `chat_message_feedback`: 106 rows (99 UP, 7 DOWN); 86 `no_api_key` failures (07-22 → 10-10); 20 forwarded (10-13 → 11-02); 8 internal rows; 79 external users; 87 `exam_id` NULL. MENTOR 70/3, SME 29/4.
- **q5** `feedback_reports`: 64 total, 3 internal, 38 NULL exam; 1 OPEN (oldest 12 d); median first reply 2.55 h.
- **q6** surveys:

  | Survey | Status | Rows | Answered | Snoozed | Dismissed |
  |---|---|---|---|---|---|
  | b2d7bebf | ACTIVE | 2,785 | 428 | 2,121 | 236 |
  | (two others) | CLOSED / DRAFT | 0 | 0 | 0 | 0 |

  b2d7bebf: 10 internal rows, 425 answered by external users. `i18n` NULL on all three.
- **q7** exams, plans and global caps: as in the Screen 5 tables. APPSC `feature_caps` = `{pyq_open: lifetime 10, pyq_reveal: daily 10}`.
- Extra queries:
  - `pyq_papers` playable counts 78 / 1040 / 24 (1 language).
  - `app_config.filter_config` = 1 APPSC override.
  - `user_profiles.ui_language` NULL ×16,249.
  - `i18n_languages` = `en` only.
  - `notification_rule` audience `{}` ×13, plus deep-link types.

Processes: none left running. The only processes were foreground `ts-node` runs (2 DTO probes, ~12 SQL runs), all exited. No server, browser or background job was started; nothing mutating was called. No `git fetch/pull/checkout`; no `.env*` read.

Files written: this report; `scratchpad/s07/settings_probe.ts`, `plan_probe.ts`, `q1.sql`…`q7.sql`.
