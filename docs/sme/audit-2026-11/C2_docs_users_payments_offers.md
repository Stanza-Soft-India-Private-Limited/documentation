# Audit C — docs ledger, agent 2 of 3: users · payments · notifications · feedback · offers · banners · exams

Read-only pass, 2026-11-03. Nothing edited, nothing run against any DB, no browser.
Paths: `B:` = `/Users/yashwanth/Desktop/projects/backend/src/` · `M:` = `upsc-document-generator/app/main.py` · `UI:` = `upsc-document-generator/admin-ui/src/` · doc = `backend/docs/sme/<doc>`.
Verdicts: **RENDERED** (UI sends/shows it) · **PROXIED-ONLY** (reaches main.py, never the UI) · **BACKEND-ONLY** (portal never calls it) · **ABSENT** (doc promises, backend lacks) · **CONTRADICTED** (doc vs code, both quoted).

---

## SME_PORTAL_API.md

| # | Doc promise (§) | Our backend | Portal proxy | Portal UI | Verdict |
|---|---|---|---|---|---|
| 1 | `GET /sme/users` list: page, limit, search, status, onboarded, platform, exam (§1.1) | `modules/sme/controllers/sme-user.controller.ts:30–41`, `dto/sme-user.dto.ts:31–103` | M:4525 passthrough M:4574–4586 forwards page/limit/search/status/onboarded/platform/exam | `services/api.ts:2086`; filters `pages/AppUsers.tsx:1136–1171`, 1398–1470 | RENDERED |
| 2 | Axis 1 — `exam=` = who they ARE (`activeExamId`, NULL ≡ upsc-cse) | `services/sme-user.service.ts` exam where (NULL fold) | upstream on passthrough; on fetch-all path re-implemented locally M:4512–4516 (`activeExamId or "upsc-cse"`) | exam badge `AppUsers.tsx:1627`; detail "No exam chosen (served UPSC)" :1039 | RENDERED |
| 3 | Axis 2 — `exam=`+`premium=true` = what they HOLD (LIVE `user_exam_entitlements` row, any exam mode) | `sme-user.service.ts:92–106` (decision D4) | **never forwarded upstream**: `_sme_users_corpus_params` M:4440–4450 drops both `exam` and `premium`, then `_sme_holds_live` M:4427–4437 over a `_sme_fetch_all` corpus (bug class 2/3) | label flips to "Holds <exam>" with axis tooltip `AppUsers.tsx:1409–1416`; LIVE-entitlement badges :1639 | RENDERED (by portal re-implementation over a full scan, not by our filter) |
| 4 | `premium=true` WITHOUT exam = person-level mirror "`user_auth` SUBSCRIBED and not expired" | `sme-user.service.ts:108` (SUBSCRIBED + not expired; trial excluded) | always local: `bool(u.get("isPremium")) != premium` M:4490–4491 — `isPremium` **includes live trial** | tooltip "person-level — premium in any exam, live trial included" `AppUsers.tsx:1414` | CONTRADICTED (doc: SUBSCRIBED-only; portal: isPremium incl. trial; labelled honestly on screen — bug class 9) |
| 5 | `platform` = "has an active device token" | `sme-user.service.ts:119–121` (`deviceTokens some {platform, isActive}`) | passthrough forwards it correctly M:4582; **fetch-all path** (any premium/premiumState/createdFrom/To set) matches `provider`/`subscriptionSource` instead M:4497–4500 | `AppUsers.tsx:1466–1468` | CONTRADICTED (same control means two different things depending on which other filter is set) |
| 6 | Response `premiumState` funnel label | `sme-user.service.ts` derivePremiumState | passthrough; local `premiumState` filter M:4492 (backend 400s on it) | `AppUsers.tsx:1034`, 1632; filter :1440 | RENDERED |
| 7 | `activeExamId` / `trialExamId` / `trialEndsAt` / `trialDaysLeft` on every row | `sme-user.service.ts:728–760` | passthrough | :1039, 1046–1056, 1633–1635 | RENDERED |
| 8 | `entitlements[]` `{examId, accessTier, status, source, expiresAt, isTrial, trialEndsAt}` | same | passthrough; CSV cell `_sme_entitlements_cell` M:4453–4465 | list shows LIVE only (:1639, title has source+expiry); detail lists all with status/source/expiry :1071–1090 | RENDERED except `isTrial`, `accessTier` (no hit in `AppUsers.tsx`) → PROXIED-ONLY for those two keys |
| 9 | 400 on unknown `?exam=` slug | `ExamsService.assertExists` | passthrough | picker sourced from `GET /sme/exams` (`components/ui/ExamSelector`) so the 400 is unreachable | RENDERED (by construction) |
| 10 | `total` is cross-exam unless `?exam=` | count shares page `where` | fetch-all path returns `len(filtered)` + `coverage` M:4563–4572 | coverage warning `AppUsers.tsx:1528–1533` | RENDERED |
| 11 | `GET /sme/users/export` (portal-only CSV) | — | M:4589 full scan, `activeExamId,trialExamId,entitlements` appended last M:4610–4617 | `exportSmeUsers` `api.ts:2095`, call `AppUsers.tsx:1310` | RENDERED (portal construct) |
| 12 | `GET /sme/users/:id` detail: `trialExtended` | `sme-user.service.ts` | M:4635 | `AppUsers.tsx:1059`, `AppUserDetail.tsx:136` | RENDERED |
| 13 | detail `whatsappExamIds` / `whatsappExamsEffective` | `sme-user.service.ts` | passthrough | `rg whatsapp admin-ui/src` = 0 hits | PROXIED-ONLY |
| 14 | detail `recentOrders[]` incl. `examId` (nullable), `isTest` | `sme-user.service.ts:~200` | passthrough | rendered from the **snapshot** route instead (`components/features/app-users/SnapshotHeader.tsx:369–373`), not from this field | PROXIED-ONLY (field) / RENDERED (equivalent elsewhere) |
| 15 | `PATCH /sme/users/:id` name/status only | `sme-user.controller.ts:58` | whitelist name,status M:4652 | `AppUsers.tsx:175`, reactivate :1257 | RENDERED |
| 16 | `POST …/deactivate`, `DELETE /sme/users/:id` | `:64`, `:71` | M:4655, 4664 (`require_admin`) | :1258, :920 | RENDERED |
| 17 | `POST …/premium` — `examId` optional, defaults `upsc-cse` (§1.6) | `dto/sme-user.dto.ts:142–153` default upsc-cse | **portal 400s without examId** M:4685–4686 | grant form sends plan/examId/amount(paise)/reference `AppUsers.tsx:254–259`; exam required, defaults to page/user exam not already held :238–249 | RENDERED (proxy deliberately stricter than doc — doc's default is the v2.40 "APPSC buyer got UPSC" footgun) |
| 18 | `DELETE …/premium?exam=` REQUIRED; `?exam=all` explicit; prefer picker from `entitlements[]` (§1.7 BREAKING) | `sme-user.controller.ts:87–108`, `sme-user.service.ts:366–373` | `exam: str = Query(...)` M:4688–4699 | per-exam radio list from LIVE entitlements + "All exams" `AppUsers.tsx:1744–1760`; second confirm for `all` :1782–1798 | RENDERED |
| 19 | revoke response `examId: null` means ALL | service | passthrough | toast echoes server's exam `AppUsers.tsx:1282–1283` | RENDERED |
| 20 | `POST …/notify` incl. `params` caps (≤10 keys, ≤40, ≤256, ≤1 KB, strings only) | `dto/sme-user.dto.ts:170+` | `_validate_notify_params` M:4716–4773 enforces the same caps | `components/features/NotificationComposer.tsx:19–26`, 77–106; send `AppUsers.tsx:579` | RENDERED |
| 21 | `GET /sme/transactions` (+`exam` incl. `none`, `includeTest`) | `sme-transaction.controller.ts:13–22`, `dto/sme-transaction.dto.ts:85,119` | M:5390–5425 forwards all 10 | `pages/Transactions.tsx:216–228` | RENDERED |
| 22 | payer columns routinely null → render "—" | service | passthrough | `Transactions.tsx:257` | RENDERED |
| 23 | `GET /sme/transactions/:id` (+`notes`) | `sme-transaction.controller.ts:26` | M:5429 | `getSmePayment` defined, never called (AUDIT §1.4) | PROXIED-ONLY |
| 24 | `GET /sme/orders` (+orderStatus, exam/none, includeTest) | `:32–43` | M:5438–5482 | `Transactions.tsx:341–353` | RENDERED |
| 25 | `reconcileHint` OK / GRANT_PENDING / UNRESOLVED_EXAM — branch on it, never on raw columns | `sme-transaction.service.ts:344–351, 382` | passthrough | chips `Transactions.tsx:387–421`; explainer + "never grant these until the exam is resolved" :359–361 | RENDERED |
| 26 | `?exam=none` filter | dto :85 | passthrough | "Unresolved exam only" `Transactions.tsx:102–124` | RENDERED |
| 27 | order money fields `offerCode` `listAmount` `storefrontCurrency` `chargedAmount` `isTest`; never sum `amount` across storefronts | `sme-transaction.service.ts:353–386` | passthrough | `Transactions.tsx:160–181` (charged in storefront ccy, strike list), offer chip :422–424, sandbox :421 | RENDERED |
| 28 | `appleTransactionId` absent (not null) on Razorpay rows | `formatOrder` `?? undefined` :358 | passthrough | `pay.razorpayPaymentId \|\| pay.appleTransactionId \|\| pay.id` `Transactions.tsx:263` | RENDERED |
| 29 | `GET /sme/orders/:id` | `:47` | M:5476 | `getSmeOrder` unused | PROXIED-ONLY |
| 30 | `GET /sme/webhooks` + `examId` per event; `?exam=` deliberately ignored | `:53–62` | M:5485–5513 — forwards page/limit/source/startDate/endDate/search, **not** exam (correct) | provider/result/examId summary `Transactions.tsx:554–582`; raw JSON `<pre>` :517–519; "cannot be filtered by exam" copy :497–498 | RENDERED |
| 31 | `/sme/blogs` CRUD (§3) | `sme-blog.controller.ts` | M:6244–6301 | `pages/Reels.tsx` (`getSmeBlogs`) | RENDERED (Reels screen; out of this group's screens, not re-audited) |
| 32 | `POST /sme/notifications/broadcast` + `exam` → `exam_<slug>` topic, `topic` echoed | `sme-notification.controller.ts:27–61` | M:4788–4802 forwards `exam` | scope picker `AppUsers.tsx:780–791`; toast prints echoed `topic` :667–668 | RENDERED |
| 33 | `POST /sme/notifications/segment` + `exam` (validated; `exam` echoed; long timeout) | `:69–76` | M:4805–4822, timeout 120 s | `AppUsers.tsx:650, 664–666` (prints `data.exam`); 120 s client timeout `api.ts:2168–2176` | RENDERED |
| 34 | "UPSC also reaches everyone who never picked an exam" warning | — | — | `AppUsers.tsx:789` | RENDERED |

**Verdict: implemented (≈29/34 rendered).** Missing/consequential: (a) row 5 — the `platform` filter silently changes meaning (device token → provider/subscriptionSource) whenever any premium/date filter is also set; (b) row 4 — "Premium" without an exam is `isPremium` (incl. trial) on the portal, SUBSCRIBED-only in the doc; (c) row 3 — the per-exam hold filter is a portal full-scan re-implementation; our own `premium=true&exam=` filter is never used.

Stale doc passages: header "Changed 2026-09-21" — nothing after 09-21 although `whatsappExamIds`/`params.exam` (10-01) landed; §4.1 points to SME_NOTIFICATIONS_API for `params` without the `params.exam` stamping.

---

## SME_TRIAL_EXTENSION_API.md

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `POST /sme/users/:id/trial-extension {days 1–365, reason ≤200}` | `sme-user.controller.ts:112–122`, `dto/sme-user.dto.ts:156–167` | M:6466–6474 | `api.ts:2874–2882`; dialog `AppUsers.tsx:396–506` (reason counter :506) | RENDERED |
| 2 | stacks on `max(now, current end)`; per-exam length from `activeExamId` | `sme-user.service.ts:407–450` | — | — | backend matches doc |
| 3 | Revival rule: flip to ACTIVE only when status is exactly UNSUBSCRIBED **and** new end is in the future; SUBSCRIBED never touched | `sme-user.service.ts:474` `shouldFlip = status==='UNSUBSCRIBED' && newEnd > now` | — | `result.statusChanged` banner `AppUsers.tsx:~432` | RENDERED; backend matches doc exactly |
| 4 | 400 when paying in ANY exam (mirror SUBSCRIBED + not expired); check `entitlements[]` before offering the button | `sme-user.service.ts:431–436` | — | button blocked when any LIVE entitlement, with explanation `AppUsers.tsx:401–403, 474` | RENDERED |
| 5 | 400 for SUSPENDED/LOCKED/ONBOARDING/INACTIVE | `:437–441` | — | `TRIAL_EXTENSION_BLOCKED_STATUSES` `AppUsers.tsx:403` | RENDERED |
| 6 | response `examId` + `trialDays` rendered together ("stacked on the 7-day X window") | `:510–520` | passthrough | `AppUsers.tsx:441–448` | RENDERED |
| 7 | `previousTrialEndsAt` (undo/audit) | yes | passthrough | `AppUsers.tsx:~438` "Previous trial end" | RENDERED |
| 8 | **Notify pairing** — backend sends no push; pair "Extend" with "Notify user" | no push in service | — | "Notify user…" opens composer prefilled with the new end `AppUsers.tsx:374–390, 456–457` | RENDERED |
| 9 | `trialExtended` on detail, `trialExamId` on every list row | yes | passthrough | :1059, :1047 | RENDERED |
| 10 | `entitlements[].isTrial` per exam | yes | passthrough | not rendered | PROXIED-ONLY |

**Verdict: implemented (9/10).** Only `isTrial` is not shown. No contradictions found; backend code (`sme-user.service.ts:407–520`) matches the revival rules word for word.

---

## SME_APPLE_IAP_API.md

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `GET /sme/transactions?source=APPLE` | as SME_PORTAL rows 21 | M:5390 | source select incl. APPLE `Transactions.tsx:43, 216–219` | RENDERED |
| 2 | `appleTransactionId`, `method: apple_iap` | `sme-transaction.service.ts:306` | passthrough | :263 id column | RENDERED |
| 3 | `order.examId` nullable, `order.isTest` on the payment row | `:~320` | passthrough | ExamBadge "No exam recorded… not the same as UPSC" `Transactions.tsx:80–89, 273`; sandbox :286 | RENDERED |
| 4 | Payment detail `notes` (`rawJws`, `storefront`) | `formatPayment(p, includePayload)` :332 | M:5429 | `getSmePayment` unused | PROXIED-ONLY |
| 5 | Orders: six fields + `reconcileHint`; `chargedAmount storefrontCurrency` shown, `amount` fallback | :353–386 | passthrough | `Transactions.tsx:160–181, 387–424` | RENDERED |
| 6 | `GET /sme/orders?source=APPLE&exam=none` bucket | dto | passthrough | `ExamFilter` + source select | RENDERED |
| 7 | "`appleOriginalTransactionId` … visible in detail" (§2.2 "Other notes", §4 "Group by `appleOriginalTransactionId` (visible in detail)") | **not emitted anywhere**: `rg appleOriginalTransactionId src/modules/sme` = 0 hits; `formatOrder` :353–386 has no such key (column exists, `prisma/schema.prisma:564`) | — | — | **CONTRADICTED / ABSENT** |
| 8 | `GET /sme/webhooks?source=APPLE`, field set `eventId…receivedAt`, `payload` is JWS not JSON | `sme-transaction.controller.ts:53` | M:5485 | Apple/Razorpay source chips `Transactions.tsx:539–543`; payload shown via `JSON.stringify(evt)` (never parsed) :517–519 | RENDERED |
| 9 | null `userId` on a processed settlement is not an error | — | — | summary hides userId; no error styling | RENDERED |
| 10 | "Which product ids / subscription groups exist → `GET /sme/exams` (`exam_plans.appleProductId`…)" (§0, §5) | `GET /sme/exams` returns exam rows only — `sme-exam.service.ts:90–96` (`exam.findMany`, no plans) | portal knows: `services/offers.api.ts:8–9` "`GET /sme/exams` carries no plans" | uses `/sme/offers/exam-plans/{exam}` → `GET /sme/exams/:id/plans` | **CONTRADICTED** (product ids live on `GET /sme/exams/:id/plans`) |
| 11 | `apple_subscriptions` state not exposed (doc admits) | none | — | — | ABSENT (documented) |

**Verdict: implemented for lists (7/11 rendered); partial overall.** Missing: (1) `appleOriginalTransactionId` — doc tells the portal to group renewals by it, but we never return it; (2) detail `notes` never shown; (3) live Apple subscription state (doc-admitted gap).

---

## SME_NOTIFICATIONS_API.md

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | Per-user notify `{title, body, type, id, params}`; `successCount:0` = no device | `sme-user.controller.ts:126`, `sme-user.service.ts:~560–580` | M:4776–4783 | `AppUsers.tsx:579` | RENDERED |
| 2 | Broadcast → `all_users` or `exam_<slug>`; response `topic` always present | `sme-notification.controller.ts:27–61` | M:4788 | toast shows `topic` `AppUsers.tsx:667–668`; audience copy names `exam_<slug>` :691, 784 | RENDERED |
| 3 | Exam topic coverage grows (devices join on next sync) | — | — | `AppUsers.tsx:691, 784–786` | RENDERED |
| 4 | Segment premium/trial/free + `exam`; `matchedUsers`, device counts, `exam` echoed | `:69–76` | M:4805 | :664–666 | RENDERED |
| 5 | Unknown exam slug → 400 on both sends | `ExamsService.assertExists` | forwarded unchanged | picker from `/sme/exams` | RENDERED |
| 6 | Deep-link `type` table (40 routable types) | — | — | `NotificationComposer.tsx` offers exactly the 40 values in §4.2 (set compared 1:1) | RENDERED |
| 7 | `params` ≤10 keys/≤40/≤256/≤1 KB, strings only | DTO | M:4716–4773 | composer caps `NotificationComposer.tsx:26`; per-type param specs :77–106 | RENDERED |
| 8 | `/sme/notifications/multi` (portal fan-out) | per-user notify | M:4827–4866 (device-level counts, `noDeviceUsers`) | `api.ts:2196–2207`; `AppUsers.tsx:651` | RENDERED (portal construct) |
| 9 | Direct SME sends bypass preferences/quiet hours/cap | true | — | not stated on the send dialog (`rg -n "quiet hours" AppUsers.tsx` = 0) | PROXIED-ONLY (rule not surfaced) |
| 10 | (missing from doc) backend stamps `params.exam` on exam-scoped broadcast/segment and on per-user sends; stamped exam overwrites a hand-typed `exam` param | `sme-notification.controller.ts:7, 58–61` (`withExamParam`); `sme-user.service.ts:29, 573` | — | composer lets an operator type an `exam` param that will be overwritten | doc STALE (documented only in DEEP_LINKS.md §6) |

**Verdict: implemented (8/10).** Missing: the bypass rule is not said on the send UI; the doc omits `params.exam` stamping.

---

## SME_NOTIFICATION_RULES_API.md

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `GET …/triggers` registry; build the form from it | `notification-engine/notification-rule.controller.ts:45`; shape `services/notification-rule.service.ts:52–67` | M:5125–5136 | config inputs generated from `configSchema` `RuleEditor.tsx:320, 480–484`; `vars` chips :302 | RENDERED |
| 2 | `kind` EVENT → no `sendAtIst` | DTO rejects | — | `RuleEditor.tsx:54, 301, 337–339` | RENDERED |
| 3 | `available:false` + `unavailableReason`, enable 400s | service :65–66 | — | `NotificationRules.tsx:378`, `RuleWorkbench.tsx:492`, `RuleEditor.tsx:463–467`; Enable disabled :620 | RENDERED |
| 4 | `transactional` flag | :64 | — | `RuleEditor.tsx:444–453` | RENDERED |
| 5 | Copy `{{vars}}`, unknown var → empty string | runner | — | unknown-var detection `RuleEditor.tsx:142–143` | RENDERED |
| 6 | Title ≤120 / body ≤500, truncated not rejected | — | — | counters + "truncated at send" `RuleEditor.tsx:145–146, 171, 190, 209` | RENDERED |
| 7 | **Per-language `copy` map (10-30 header)** — each recipient gets copy in their UI language | `dto/notification-rule.dto.ts:20–29, 121, 192` (`IsLanguageMapOf`) | `copy` forwarded whole M:5095–5097 | portal writes only `{en:{…}}` `services/api.ts:2385–2387`; reads `en` :2371–2382 | PROXIED-ONLY (no way to author/see `hi` etc.) |
| 8 | Deep link + `params` caps; engine adds `rid` | — | `_rule_body` validates params M:5104–5118 | `RuleEditor.tsx` deepLink | RENDERED |
| 9 | Audience `tiers` `platforms` `activeWithinDays` `inactiveForDays` | DTO | forwarded | `RuleEditor.tsx:636–659` (+day inputs) | RENDERED |
| 10 | `examId` on the rule, null = every exam; upsc-cse also matches NULL | — | key-membership filter keeps `examId:null` M:5104–5111 | `RuleEditor.tsx:309–327`, warning :730 | RENDERED |
| 11 | CRUD `GET ?trigger&exam&enabled`, `POST`, `PATCH` (merge), `DELETE` = archive | controller :75–150 | M:5177–5271; `isEnabled`/`rolloutPercent` refused on PATCH M:5246–5252 | list `NotificationRules.tsx`; filters applied **client-side** :132–139 | RENDERED (list `?exam=`/`?trigger=` upstream params PROXIED-ONLY) |
| 12 | `GET /:id` | :95 | M:5213 | `getNotificationRule` unused | PROXIED-ONLY |
| 13 | `triggerKey` immutable → "duplicate as new rule" | — | POST-only | `NotificationRules.tsx:421`, `RuleEditor.tsx:433` | RENDERED |
| 14 | Preview: `matchedUsers`/`afterPrefs`/`afterCaps`/`suppressed`/`sample`/`skippedReason`/`error`; **null ≠ 0** | :111 | M:5274–5286 | funnel `RuleWorkbench.tsx:95–145`; `nfmt(null)='—'` `shared.tsx:48` | RENDERED |
| 15 | Test ≤10 users; `deviceCount:0` warning; `wouldSuppress` reported not honoured | :120 | max 10 M:5291–5312 | `RuleWorkbench.tsx:181–260` | RENDERED |
| 16 | Enable `{rolloutPercent}` / disable; default 0%; deterministic bucketing | :131–145 | M:5318–5351 | stepper + reach estimate `RuleWorkbench.tsx:380–445, 552–553` | RENDERED |
| 17 | Settings: kill switch, shadow, IST quiet hours, `maxPerUserPerDay` | `dto/notification-rule.dto.ts:288–298` | M:5139–5174 (validates cap ≤5) | `GuardrailsPanel.tsx:91–201` | RENDERED |
| 18 | `maxPerUserPerDay` "**clamped server-side to 0–5. A value outside that is corrected, not obeyed**" (§7) | DTO `@Min(0) @Max(5)` → **400** (`notification-rule.dto.ts:293–298`); service also clamps defensively (`notification-settings.service.ts:72–99`) | proxy 400s >5 M:5165–5173 | input `max=` `GuardrailsPanel.tsx:162` | **CONTRADICTED** (doc §7 says corrected; same doc's error table and the DTO say 400) |
| 19 | Runs + analytics `{sent, tapped, tapRate}`; `tapped` ≠ `isRead` | :153–166 | M:5354–5383 | `RuleWorkbench.tsx:286–312`; no `isRead` anywhere on this screen | RENDERED |
| 20 | `isLiveNow` derived — "Use it" (page guide §4) | `notification-rule.service.ts:534` | passthrough | portal re-derives from `isEnabled`/`rolloutPercent` `notification-rules/shared.tsx:14–19` | PROXIED-ONLY (field ignored; equivalent logic) |
| 21 | Language variants "stored but not populated — every rule is English today" (§2, §9) | per-language map live since 10-30 (row 7) | — | UI repeats it: `RuleEditor.tsx:513` "English only today. Other languages are stored but not populated, so every user sees this text." | **CONTRADICTED** (doc body vs its own 10-30 header and the DTO; UI copy now false) |
| 22 | Per-exam entitlement targeting impossible; `pib_today` unavailable | — | — | — | documented gaps |

**Verdict: implemented (≈17/22 rendered).** Missing: per-language copy authoring (row 7) with false "English only" copy on screen (row 21); `isLiveNow` ignored (row 20).

---

## SME_NOTIFICATION_RULES_PAGE_GUIDE.md

| # | Brief item | UI evidence | Verdict |
|---|---|---|---|
| 1 | Enable ≠ send; ramp stepper 5→25→50→100 with "N% of M ≈ K people" | `RuleWorkbench.tsx:380, 436–445` | RENDERED |
| 2 | Form built from `/triggers` `configSchema`, never hardcoded | `RuleEditor.tsx:320, 480–484` | RENDERED |
| 3 | `kind` EVENT hides send time | `RuleEditor.tsx:301, 337–339` | RENDERED |
| 4 | `available:false` read-only + Enable disabled | `RuleWorkbench.tsx:492, 620` | RENDERED |
| 5 | `examId` select by hand, "All exams" first = null, UPSC-NULL warning beside the select (not a tooltip) | `RuleEditor.tsx:309–327, 730` | RENDERED |
| 6 | `{{variable}}` chips beside title/body | `RuleEditor.tsx:113–134` (insert at caret) | RENDERED |
| 7 | "`/triggers` now also carries `sampleVars`" | **not in the response**: `listTriggers()` `notification-rule.service.ts:52–67` emits no `sampleVars` (only used internally by `rule-runner.service.ts:447`) | **CONTRADICTED**; portal falls back to local samples and says so `notification-rules/shared.tsx:94–96`, `RuleEditor.tsx:237–238` |
| 8 | Live counters, title 120 / body 500 | `RuleEditor.tsx:145–209` | RENDERED |
| 9 | Preview before Enable is reachable | `RuleWorkbench.tsx:441–445, 620` (`disabled={!previewRun …}`) | RENDERED |
| 10 | Preview **as a funnel**, exam chip beside it, re-run on exam change | `RuleWorkbench.tsx:51–66, 102–145, 503, 524` (`previewStale`) | RENDERED |
| 11 | null counts → "not applicable", never 0, no reach estimate from null/EVENT/error | `RuleWorkbench.tsx:102, 436–441` | RENDERED |
| 12 | Keep the reach stepper for EVENT rules, no people-count | `RuleWorkbench.tsx:439` (`isEvent ? null`) | RENDERED |
| 13 | Label `matchedUsers` as "the audience", `after*` as "would go out now" | `RuleWorkbench.tsx:122–126` | RENDERED |
| 14 | `deviceCount:0` warning; `wouldSuppress` labelled "a real run would skip…" | `RuleWorkbench.tsx:237–260` | RENDERED |
| 15 | List: one status (Off / Enabled 0% / Live N%), grouped by trigger, exam column | `notification-rules/shared.tsx:10–45`; `NotificationRules.tsx:72, 150, 336` | RENDERED |
| 16 | Runs: SHADOW rows are success; show `mode` | `RuleWorkbench.tsx:340–357` | RENDERED |
| 17 | `tapped` never next to `isRead`; 2.0-only coverage caveat | `RuleWorkbench.tsx:293–312` ("app 2.0+ installs only") | RENDERED |
| 18 | `tapped` "stamped by `POST /notification-preferences/tapped`" (§5) | real route is `POST /notifications/tapped`: `notification-engine/notification-preferences.controller.ts:59` `@Controller('notifications')`, :97 `@Post('tapped')` (SME_NOTIFICATIONS_API §4.2b has it right) | **CONTRADICTED** |
| 19 | Settings: "clamped 0–5 server-side; the API 400s above 5" | DTO 400s (row 18 above) | half-CONTRADICTED (clamp vs 400 in one sentence); UI enforces `max` `GuardrailsPanel.tsx:162` |
| 20 | Say transactional bypass on the settings page | `GuardrailsPanel.tsx:201` (kill-switch dialog says transactional normally bypass) | RENDERED |
| 21 | No send-now, no triggerKey editor, no hard delete, no per-user prefs | none present | RENDERED (absence honoured) |
| 22 | tapRate null vs 0 | `RuleWorkbench.tsx:300` shows "not yet available" when `tapped === 0` even with `sent > 0` — a real 0% is rendered as unknown | minor (bug class 10, inverted) |

**Verdict: implemented (19/22).** The two guide-side errors (`sampleVars`, tapped route) are doc faults, not portal faults.

---

## SME_FEEDBACK_API.md

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | Reports list filters type/status/categoryKey/platform/tier/contextType/userId/search/from/to/page/limit/**exam** | `sme-feedback.controller.ts`, `services/sme-feedback.service.ts:62–89` | M:6557–6586 forwards all 13 | `components/features/feedback/ReportsTab.tsx` | RENDERED |
| 2 | `?exam=none` = NULL exam_id ("not recorded", never UPSC) | :62–68 | forwarded | "No exam recorded" option `ReportsTab.tsx:56–57, 431`; row chip "not recorded — not UPSC by default" :349 | RENDERED |
| 3 | Row `examId` | :119 | passthrough | `ReportsTab.tsx:349` | RENDERED |
| 4 | Detail `context.examIds` (content axis) vs `examId` (filed-from axis); omitted not `[]` | :335–378 | passthrough; plus portal-composed `/target` M:6707 | `ReportTargetCard.tsx:75–99, 212–214` (mismatch note) | RENDERED |
| 5 | `SIMULATION_QUESTION` → "read it from `GET /sme/content/simulations/:id`" | **no such route** (`sme-content-ingest.controller.ts` has list/create/patch/assign/questions-bulk only — AUDIT §2.3) | portal builds detail from the keyed list M:8436–8444 | — | **CONTRADICTED** |
| 6 | Status / reply (PLANNED/SHIPPED only on FEATURE) | controller | M:6820–6839 | ReportsTab | RENDERED |
| 7 | Chat summary incl. `difyForwardFailures` alert | :215 | M:6844 | `ChatFeedbackTab.tsx:80–89` | RENDERED |
| 8 | Chat messages rows incl. `rating`, `examId`, `updatedAt` (thumb can change since 2.1) | :245–263 returns raw Prisma rows; `rating` is enum `UP`/`DOWN` (`prisma/schema.prisma:2239–2242`; doc's own capture shows `"rating": "UP"`) | M:6852–6861 | **`m.rating === 'up'` `ChatFeedbackTab.tsx:188` — never true, every row renders a thumbs-DOWN even under the "up" filter**; `examId` and `updatedAt` not rendered (only `createdAt` :205) | **CONTRADICTED** (render bug) + PROXIED-ONLY (`examId`, `updatedAt`) |
| 9 | `rating` defaults `down`; label which side | — | — | default 'down' + select `ChatFeedbackTab.tsx:136, 154` | RENDERED |
| 10 | Config GET/PUT chipSets/survey/limits incl. `listWindowDays` | — | M:6869–6884 | `FeedbackConfigTab.tsx:124` | RENDERED |
| 11 | Survey create/edit incl. **`targetExams`** (empty = every exam; `"*"` rejected; upsc-cse incl. never-picked) | `services/sme-survey.service.ts` | whitelists incl. `targetExams` M:6902–6905, 6976–6979 | `SurveyBuilderModal.tsx:229, 277, 289, 398–408` (active exams only, no `*`, empty=every exam copy) | RENDERED |
| 12 | Per-language survey `title`/`description`/`questions` maps; GET adds `…ByLanguage` (10-30) | survey DTO | forwarded (same keys) | `rg ByLanguage admin-ui/src` = 0 | PROXIED-ONLY |
| 13 | DRAFT→ACTIVE freezes questions (409) — confirm before | — | M:6948 | `SurveysTab.tsx:181` close dialog; activation confirm (not re-read) | RENDERED |
| 14 | Detail `counts {answered, snoozed, dismissed}`; dashboard-vs-inbox dismissal note | — | passthrough | `SurveyResultsView.tsx:248, 283–313` | RENDERED |
| 15 | List `responseCount` — doc: "did anyone answer? (`responseCount`)" | `sme-survey.service.ts:254` `_count.responses` counts **every status** (ANSWERED + SNOOZED + DISMISSED_PERMANENT; `SurveyResponse.status` `schema.prisma:2350`) | passthrough | column labelled "Reached = answered + snoozed + dismissed" `SurveysTab.tsx:105` (correct), but close dialog says "The N responses already collected" :181 (over-states) | **CONTRADICTED** (doc implies answers) |
| 16 | Notify `{title, body}` → cohort incl. `targetExams`, excludes answered/dismissed | — | M:6953–6962, 120 s | `SurveyResultsView.tsx:203` | RENDERED |
| 17 | Results `breakdown=tier\|platform\|exam`; `examNullCount`, `examNullBucketedAs`, `examNote` beside the chart | `:511–565` | M:6966 forwards `breakdown` | toggle `SurveyResultsView.tsx:323–330`; caveat :259–262, 349–362; per-group note :139–148 | RENDERED |
| 18 | `npsScore 0` on zero answers → gate on `answered > 0` | — | — | `SurveyResultsView.tsx:49–78` | RENDERED |
| 19 | `GET …/responses` rows carry `exam` | service | M:6984 | `getSmeSurveyResponses` unused | PROXIED-ONLY |
| 20 | CSV `export.csv`, `exam` as LAST column | `:744` | `_sme_proxy_raw` M:6988–6997 (no params → no merge bug) | download `api.ts:3269–3271`, `SurveyResultsView.tsx:267` (file passed through untouched) | RENDERED |
| 21 | "On production today there is no `mentor` key … `difyForwardFailures` equals `total`" (§2, §7) | config, not code | — | UI keeps a generic alert | doc likely STALE (memory: `DIFY_APP_KEYS` set 2026-10-12) — UNVERIFIED, draft SQL: `SELECT count(*) FILTER (WHERE dify_forwarded) AS ok, count(*) AS total FROM chat_message_feedback WHERE created_at >= '2026-10-12';` |

**Verdict: implemented (14/21 rendered).** Most consequential: (1) the chat-feedback table renders every rating as a thumbs-down (`'up'` vs `UP`); (2) per-language survey content and the chat row `examId` are invisible; (3) the doc's `responseCount` meaning.

---

## SME_OFFERS_API.md

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | List `?status&exam`, `{data,total}` unpaginated, exam strict 400 | `sme-offer.controller.ts:36` | M:7344–7364 forwards status, exam | `api.ts:4166–4168`; `Offers.tsx:2044, 2166` | RENDERED |
| 2 | `examId` on every row; exam column; two LIVE in different exams is fine | `sme-offer.service.ts:~1330` | passthrough | `Offers.tsx:1797`; overlap check scoped per exam :525 | RENDERED |
| 3 | `isActiveNow` derived | service | passthrough | per-exam "showing now" (CLAUDE.md:309) | RENDERED |
| 4 | **`examId` REQUIRED on create, no default, first field** (§3 BREAKING) | `sme-offer.service.ts:878` | whitelist M:7337–7341 | `form.examId: ''` :320; "Exam is required" :490; selectable exams :921 | RENDERED |
| 5 | `examId` editable, tri-state; re-checks floor + window in target exam | service | forwarded | sent only when changed `Offers.tsx:928, 976` + warn | RENDERED |
| 6 | `requiresCode` (default false in API) | `:189` `?? false` | forwarded | toggle "Only reachable with the share link" `Offers.tsx:1104–1110`; **new-campaign default ON** :327 (per CAMPAIGN_FORM_GUIDE §2.1) | RENDERED |
| 7 | `heroImageUrl`, `bannerImageUrl` (5:2, 1500×600, `folder: banners`) | DTO :233–243 | forwarded | `Offers.tsx:1146–1170`, upload :548 | RENDERED |
| 8 | **`priceInPaise` is the only money**; below that exam's standard; yearly-only | `:1180–1215` | forwarded inside `plans` | "Amount actually charged (₹)" → paise `Offers.tsx:666–669`; floor vs exam plans `offerPricing.ts:127–131`; ₹0 hard confirm `Offers.tsx:919, 1448` | RENDERED |
| 9 | Exam's list price shown beside the charge | `GET /sme/exams/:id/plans` | M:7377–7385 (`/sme/offers/exam-plans/{exam}`) | `offers.api.ts:28–37`; `Offers.tsx:190–198, 619` | RENDERED |
| 10 | `appleOfferCode` (`^[A-Z0-9]{1,64}$`, hard 400) / `appleProductId` (must be that exam's product) | DTO | forwarded | uppercase-as-typed `Offers.tsx:693–697`; product hint + "use this exam's product" :703–718; App Store readiness :929 | RENDERED |
| 11 | `bonusDays` must be 0 (non-zero = 400) | `:1211` | forwarded | control removed; payload hard-codes `bonusDays: 0` `Offers.tsx:458` | RENDERED |
| 12 | `plans`/`content` MERGE on PATCH; `styles`/`socialProofAvatars` replaced wholesale | `:~300–310` | forwarded | send complete `styles`, null to clear `Offers.tsx:403–434`; fresh read before edit :870–897 | RENDERED |
| 13 | `content.styles` (`#RRGGBB`, bold) on 9 fields; `socialProofAvatars` ≤5 https | sanitiser | forwarded | `Offers.tsx:346–347, 384–434, 500–503`; contrast warning :744–810 | RENDERED |
| 14 | The two "Prices are display strings — the buy flow still charges base pricing" banners must go | — | — | **both gone**: `rg -i "configured base\|base pricing"` = 0 hits; replaced by true amber note "Price is the text shown on screen. Amount actually charged is the money…" `Offers.tsx:1185–1195` and section header "Shown on screen (text only)" :647; recorded in portal `CLAUDE.md:370` | RENDERED (removal honoured; remaining copy is true) |
| 15 | **`contentByLanguage`** on detail/list; `content` accepts a per-language map (10-30) | `sme-offer.service.ts:1348–1350`, DTO :280 | `content` forwarded whole | `rg contentByLanguage admin-ui/src` = 0 | PROXIED-ONLY |
| 16 | Pause (reversible) vs End/Archive (terminal) made obvious | `:123–146` | M:7448–7482 | `Offers.tsx:1543–1544, 1680–1682` | RENDERED |
| 17 | Hero/banner/exam-logo upload via `POST /sme/media/upload-url` | `sme-media.controller.ts` | M:7801–7816 (filename, contentType, folder) | `uploadSmeMedia` `Offers.tsx:548, 1477` | RENDERED |
| 18 | `GET /paywall/banner` rule; `requiresCode` ignored for banners | app-side | — | helper text says "or tap the dashboard banner" when a banner is set `Offers.tsx:1110` | RENDERED |
| 19 | Preview `?code=&preview=1` + api key (device QA) | app-side | — | not a portal feature (needs key on device) | n/a |
| 20 | "This exists because the portal's offer form has no `priceInPaise` / `appleOfferCode` / `appleProductId` input" (§4) | — | — | the form has all three (rows 8, 10) | doc STALE |
| 21 | "Source the product id from `GET /sme/exams`" (§7.1 "App Store Connect groups") | `GET /sme/exams` has no plans (`sme-exam.service.ts:90–96`) | — | portal reads `/exams/:id/plans` | **CONTRADICTED** |
| 22 | `GET /sme/offers/:id` | `:48` | M:7388 | used for fresh read before edit `Offers.tsx:870` | RENDERED |

**Verdict: implemented (≈19/22).** Missing: `contentByLanguage` (no per-language campaign copy in the portal). Doc faults: §4 stale sentence, §7.1 wrong product-id source.

---

## SME_CAMPAIGN_ABANDONED_API.md

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `GET /sme/offers/:id/abandoned` signal/from/to/minAppVersion/page/limit | `sme-offer.controller.ts:55`; service :465–630 | M:7487–7536: validates signal ∈ {all,view,order,tap}, default `minAppVersion=1.8`, forwards from/to only when set | `api.ts` `getSmeOfferAbandoned`; `OfferAnalytics.tsx:303–322` | RENDERED |
| 2 | **`signal` semantics** — table reads as a filter ("`order` (certain only)", "`tap` (inferred only)"); "users counted at furthest stage, signals partition the audience" | a user's label is computed only from the CTEs the request asked for (`wantOrder/wantTap/wantView` :491–493; CASE :579–583), so `signal=view` returns every viewer labelled VIEW incl. people who also ordered — it **re-labels, it does not subset**; sums across separate calls over-count | proxy docstring documents the re-label trap M:7506–7511 | fetches `all` + `view` once each, dedupes by user keeping the furthest stage `OfferAnalytics.tsx:326–361` | **CONTRADICTED** (doc implies subset filter; portal correctly works around it) |
| 3 | `minAppVersion` floors iOS TAP only; default 1.8; `0` disables | :472–473, 529–533 | default "1.8" M:7493 | `OfferAnalytics.tsx:580` control | RENDERED |
| 4 | `from`/`to` clamp INTO the window, never widen | service | forwarded only when set | IST date control `OfferAnalytics.tsx:581, 733–751`; window shown :686 | RENDERED |
| 5 | `campaign.examId` in the response — no extra GET | service | passthrough | `OfferAnalytics.tsx:795–797` | RENDERED |
| 6 | `funnel.*` campaign-wide; `pageCounts` page-local, never a headline | service :640–700 | passthrough | funnel from `funnel.*` `OfferAnalytics.tsx:400–445`; `pageCounts` unused as headline | RENDERED |
| 7 | `amountPaise`/`listAmountPaise` null on TAP → dash, never ₹0 | — | — | `formatPaise(null)` dash (portal CLAUDE.md:406) | RENDERED |
| 8 | `platform` null = Unknown, never "web" | :598 | — | `platformLabel` `OfferAnalytics.tsx:123–124, 1086` | RENDERED |
| 9 | Exclusions (converted, granted, `is_test`, suspended) | service | — | "N excluded (already converted)" bridge `OfferAnalytics.tsx:630–639, 941–947` | RENDERED |
| 10 | **VIEW "deploy-forward from 2026-08-20 … Every other signal is complete for the full campaign history"** (§1) | VIEW reads raw `api_usage` (:547) which is purged at **30 days** (`api-usage-rollup.service.ts:30, 126–127`); TAP reads `auth_events` (:522) purged at **90 days** (:32) | — | deploy-forward badge only `OfferAnalytics.tsx:867` | **CONTRADICTED** — for any campaign window older than 30 d (VIEW) / 90 d (TAP) those rows silently vanish; only ORDER is permanent |
| 11 | `/purchased` companion; attribution `orders.offer_code` only; `capturedPaise:null` = warning chip | `sme-offer.controller.ts:71` | M:7539–7573 | Band 4 `OfferAnalytics.tsx:377–390, 1222–1238` | RENDERED |
| 12 | Pair with notify, no send on this endpoint | — | `/sme/notifications/multi` | chase list notify for certain rows only `OfferAnalytics.tsx:624–626` | RENDERED |
| 13 | `attempts > 1` → check incidents | — | — | not surfaced as a cue (not re-verified) | PROXIED-ONLY |

**Verdict: implemented (11/13).** Doc faults are the consequential ones: retention (row 10) and signal semantics (row 2).

---

## SME_CAMPAIGN_FORM_GUIDE.md

| # | ADD / CHANGE / REMOVE item | UI evidence | Verdict |
|---|---|---|---|
| 1 | REMOVE list-page "Prices are display strings…" banner | 0 hits for "configured base / base pricing"; portal `CLAUDE.md:370` | RENDERED (removed) |
| 2 | REMOVE PLANS-section banner, replace with §3.1 helper | `Offers.tsx:1185–1195` | RENDERED |
| 3 | ADD `requiresCode` toggle, default ON, helper mentions banner caveat | `Offers.tsx:327, 1104–1110` | RENDERED |
| 4 | ADD `priceInPaise` per plan, warn blank-vs-price, mismatch, ₹0 hard confirm, compare vs SELECTED exam | `offerPricing.ts:127–136, 202–226`; `Offers.tsx:619, 919, 1448` (input is rupees → paise, not raw paise) | RENDERED |
| 5 | ADD `appleOfferCode` (uppercase as typed) | `Offers.tsx:693–697` | RENDERED |
| 6 | ADD `appleProductId` | `Offers.tsx:703–718` | RENDERED |
| 7 | REMOVE Bonus days | no input; `bonusDays: 0` hard-coded :458 | RENDERED |
| 8 | Rename HERO IMAGE → IMAGES; banner upload 5:2 1500×600 | `Offers.tsx:1146–1170` | RENDERED |
| 9 | Colour + bold on 9 fields, whole `styles` object every save, reset-to-default, contrast warning <4.5:1 | `Offers.tsx:384–434, 744–810, 935–938` | RENDERED |
| 10 | `socialProofAvatars` ≤5 | `Offers.tsx:347, 413, 503` | RENDERED |
| 11 | Preview: remove SEC cell, `:` separator | `OfferPaywallPreview.tsx:37–38` | RENDERED |
| 12 | `examId` first field, no prefill, editable with warning, list exam column + `?exam=` | `Offers.tsx:320, 490, 921–928, 976, 1797, 2044, 2166` | RENDERED |

**Verdict: implemented (12/12).** Guide is fully consumed; its §0 ("form has no `priceInPaise`…") is now history.

---

## SME_CAMPAIGN_ANALYTICS_PAGE_GUIDE.md

| # | Brief item | UI evidence | Verdict |
|---|---|---|---|
| 1 | Band 1 verdict sentence + ≤3 stats + exam chip from `campaign.examId` | `OfferAnalytics.tsx:784–840` | RENDERED |
| 2 | Band 2 funnel Viewed→Joined→Paid, units = users, orders as subtitle, "since 20 Aug" badge while `viewedIsDeployForward`, clickable drops | `:400–480, 855–867` | RENDERED |
| 3 | Band 3 ONE merged table with `signal` chip + filter; default VIEW+ORDER+BOTH, TAP off | `DEFAULT_SIGNALS = ['ORDER','BOTH','VIEW']` :222; chips :220–225, 884–901 | RENDERED |
| 4 | Band 4 `/purchased` + `capturedPaise` warning chip; empty state first | `:377–390, 1222–1238` | RENDERED |
| 5 | Internal-accounts toggle default ON with hidden count | `INTERNAL_DOMAINS = ['stanzasoft.com','designasylum.in']` :92–97; toggle default true :584; "N internal hidden" :954 | RENDERED, but the domain list is incomplete vs the audit's internal set (`@stanzasoft.ai`, `@prepmonkey.*` missing) and applies to the chase list only. ⚠️ Memory note "Internal-domain filter DECLINED by user — do not propose again" (offer campaigns) — reported, not proposed |
| 6 | State the queried window from `window.from/to` | `:686` | RENDERED |
| 7 | Bridge every exclusion ("2 excluded — already converted") | `:630–639, 941–947` | RENDERED |
| 8 | Never sum ORDER + TAP | certain vs inferred kept apart; notify only on certain :624–626 | RENDERED |
| 9 | `/offers` list: exam column, `?exam=`, no warning for two LIVE in different exams; remove false banner | `Offers.tsx:525, 1797, 2044, 2166` | RENDERED |
| 10 | Drop the extra `GET /sme/offers/:id` used only for the exam | only two queries on the page (`fetchAbandonedCorpus`, `fetchPurchased`) `OfferAnalytics.tsx:595–601` | RENDERED |
| 11 | `viewed` 0 before 2026-08-20 | badge | RENDERED — but the guide (like row 10 of the abandoned doc) never mentions the 30-day `api_usage` purge, so the badge's "since 20 Aug" is wrong for any window older than 30 days |

**Verdict: implemented (11/11 brief items present).** One inherited doc error (VIEW retention).

---

## SME_BANNERS_API.md

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | List `?exam&status`, `{data,total}`, exam STRICT 400 | `sme-banner.controller.ts` | M:7695–7714: forwards exam; validates status itself (doc: upstream leaves it unvalidated) | `getSmeBanners()` called **without** params `pages/Banners.tsx:40`; grouped/filtered client-side :132, 190 | RENDERED (list) / PROXIED-ONLY (`?exam=` upstream filter) |
| 2 | `isLiveNow` derived — use it | service | passthrough | `Banners.tsx:86, 190`; `BannerCard.tsx:126` | RENDERED |
| 3 | Audience **tier × platform × age** (paid/trial/lapsed/free; inclusive whole-day bounds; empty array not authorable) | DTO | whitelist M:7671–7675 | tier/platform toggles + "pick at least one" `BannerEditor.tsx:52, 82–85, 106–143`; tier hints `bannerFormat.ts:78–79`; age sentence `BannerCard.tsx:87` | RENDERED |
| 4 | `trialDays = 0` exam → targeting only `trial` reaches nobody | — | — | warning `BannerCard.tsx:106`, `BannerEditor.tsx:222–224`, `Banners.tsx:202–205` | RENDERED |
| 5 | Destination allow-list `https://app.\|go.prepmonkey.com/open/…` | DTO 400 | — | `destinationError` `services/banners.api.ts:122–127`; hint `BannerEditor.tsx:439` | RENDERED |
| 6 | `imageUrl` https only; 5:2 / 1500×600 | DTO | — | `BannerEditor.tsx:396` (+ upload `folder: banners`) | RENDERED |
| 7 | `status` rejected on create/update | DTO | `_refuse_banner_status` M:7679, 7736, 7753 | — | RENDERED |
| 8 | Status machine DRAFT→LIVE↔PAUSED→ENDED (terminal), activate refused when window closed | service | M:7760–7798 | activate/pause `BannerCard.tsx:52`; end `Banners.tsx:66` | RENDERED |
| 9 | PATCH merge, explicit null clears age bounds; only changed keys sent | service | forwarded | `BannerEditor.tsx:136–143` (diff-only body) | RENDERED |
| 10 | priority −1000…1000; campaign banner injected at priority 0 | — | — | priority bounds `BannerEditor.tsx:114–115`; campaign row "priority 0 (fixed)" `BannerCard.tsx:244–252`; carousel preview :237–248 | RENDERED |
| 11 | `GET /sme/banners/:id` | yes | M:7726 | `getSmeBanner` unused | PROXIED-ONLY |

**Verdict: implemented (9/11).** Nothing consequential missing; `?exam=` strictness is never exercised because the page loads everything and filters locally.

---

## SME_EXAMS_API.md (all sections incl. §7–§12)

| # | Doc promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `GET /sme/exams` (all, display order) | `sme-exam.controller.ts:25`; `sme-exam.service.ts:90–96` | M:7084 | `exams.api.ts:142`; `ExamList.tsx`; also every `ExamSelector` | RENDERED |
| 2 | `GET /sme/exams/:id` + `effectiveFeatureFlags` / `effectiveFeatureCaps` — display effective, edit raw | `:51`; service :104–113 | M:7211 | `FeaturesTab.tsx:71, 87, 107, 223` | RENDERED |
| 3 | `content-counts` with `visible / shared / exclusive / enabled`; `exclusive: 0` = forgotten tags | `:67` | M:7252 | `ContentCountsTab.tsx:17–76` ("enabled module has nothing authored", "empty tab", "only shared content") | RENDERED |
| 4 | `POST /sme/exams` id/displayName/shortName/examDate/enabledModules/accessTier/appleSubscriptionGroupId/isActive/sortOrder; create inactive | `:79` | whitelist M:7116–7119 | `CreateExamModal.tsx:16–36` sends `isActive:false`, id, names, examDate, sortOrder (tier/group/modules set later in Settings) | RENDERED |
| 5 | `PATCH` any field but id; upsc-cse cannot be deactivated | `:90` | M:7221–7249 (id refused; upsc-cse guard; key-membership filter) | Activate/Deactivate `ExamList.tsx:35–45, 161–172` | RENDERED |
| 6 | `accessTier` free/paid; `paid` rejected until a plan exists | service | forwarded | radio + "paid blocked, no active plan" `SettingsTab.tsx:96, 185–190` | RENDERED |
| 7 | `appleSubscriptionGroupId` — one group per exam | — | forwarded | `SettingsTab.tsx:212` | RENDERED |
| 8 | Plans `GET/POST /:id/plans` (planId, planType, priceInPaise, **strikePriceInPaise** > price, **appleProductId**, **razorpayPlanId** globally unique); upsert MERGES; `DELETE` deactivates | `:103–133` | M:7262–7316 (whitelist M:7124–7126, planType check) | `PlansTab.tsx:21–115, 215`; sends only changed keys :59–65; warns before clearing a store id :65; retire `exams.api.ts:186` | RENDERED |
| 9 | §5 "`POST /sme/notifications/broadcast` is a **topic** blast and **cannot be segmented by exam**" | broadcast takes `exam` → `exam_<slug>` since 2026-09-21 (`sme-notification.controller.ts:27–61`) | M:4799–4801 | `AppUsers.tsx:780–791` | **CONTRADICTED** (stale §5) |
| 10 | §7 feature flags via **`GET /sme/exams/feature-registry`**, render toggles from it (41 keys) | `:35`; registry `common/constants/feature-registry.ts` (41 labelled entries — matches) | M:7173–7180 | `getFeatureRegistry` `FeaturesTab.tsx:13, 55`; label/parent/"off via parent"/alwaysOn :182–200 | RENDERED |
| 11 | `featureFlags` partial, `true` removes override, parent-off ⇒ children-off, `dashboard` 400, unknown key 400 | service | forwarded | pending-effective calc `FeaturesTab.tsx:66–70`; alwaysOn not togglable :197 | RENDERED |
| 12 | `enabledModules` is a derived mirror — write flags | service | still whitelisted M:7117 | UI writes `featureFlags` only | RENDERED |
| 13 | §8 `featureCaps` `{type, limit}` per cap key; 4 types; `null` = use global; malformed = 400 | `sme-exam.service.ts:447–449` (closed key list) | `_validate_exam_config` (cap types M:7127) | `FeaturesTab.tsx:18, 83–117, 223–266` ("Use global" :266; lifetime_per_subject outside content_doc warning :257) | RENDERED |
| 14 | Switching a cap TYPE on a live exam resets counters | — | — | (not verified that the UI warns) | PROXIED-ONLY (not seen) |
| 15 | **`pyq_open` cap** (per-account free question set, APPSC) — in no SME doc | valid key: `modules/app-config/app-config.service.ts:51, 58–69` (`OPTIONAL_FEATURE_CAP_KEYS`); enforced `common/guards/usage-quota.guard.ts:161–170`, `modules/pyq/pyq.controller.ts:104, 386–403`; **no registry `quota` link** | forwarded | cap rows = effective ∪ stored ∪ registry quotas `FeaturesTab.tsx:85–91` → `pyq_open` appears **only if an exam already stores it**, and is then labelled "**no feature switch (legacy key for older app builds)**" :232 — false for `pyq_open`; no way to add it to an exam that lacks it | doc ABSENT; UI partially editable with false copy |
| 16 | §9 `trialDays` 0–90, new signups only, `trialPolicy` append-only history | service :278–322 | forwarded (key-membership keeps `0`) | input 0–90 + confirm `SettingsTab.tsx:75–95, 221–224`; history list :228–232; list chip "no trial / N-day trial" `ExamList.tsx:153` | RENDERED |
| 17 | §9 "`upsc-cse` was left untouched at 14"; §12 invariant "the trial is 14 days from the epoch entry"; §2 example `trialDays: 14` | upsc-cse cut to 3 on 2026-09-25 (doc's own §4) | — | — | doc STALE (internally inconsistent) |
| 18 | §10 `logoUrl` via upload `folder: exam-logos`; null clears; 512×512 | media controller | M:7801 | `SettingsTab.tsx:120, 160–168` | RENDERED |
| 19 | §11 `paywallContent` strings, `highlightStyle` STRING enum, `benefits` ≤12, `extraComparisonRows` (feature/free/premium; `label` alias) merged + sanitised | service | forwarded | `SettingsTab.tsx:15, 39–52, 88–91, 265–292` (legacy `label` read :52) | RENDERED |
| 20 | Generated Free-vs-Premium table; writing any of paywallContent/flags/caps flips UPSC off the legacy table | app side | — | "the table is generated…" `SettingsTab.tsx:292`; legacy-flip warning `shared.tsx:121–123`, `SettingsTab.tsx:247–252`, `FeaturesTab.tsx:155–159` | RENDERED |
| 21 | **`paywallContentByLanguage`** / per-language `paywallContent` (10-30) | `sme-exam.service.ts:341–353` | forwarded whole | `rg ByLanguage admin-ui/src` = 0 | PROXIED-ONLY |
| 22 | §12 server enforcement (403 `FEATURE_DISABLED`, gated routes, daily tasks, `/quota/begin`) | yes | — | app-side; nothing to render | n/a |
| 23 | §3 tagging / §6 launch checklist (content-counts, filter-config per exam) | — | — | ContentCountsTab; FilterConfig page (other agent) | RENDERED |

**Verdict: implemented (≈17/21 applicable).** Missing: `pyq_open` (undocumented, not addable, mislabelled when present); per-language paywall copy; doc §5/§9/§12 stale.

---

## Summary

| Doc | Verdict | Rendered / rows | Missing or wrong (count) | Top items |
|---|---|---|---|---|
| SME_PORTAL_API.md | implemented | ~29/34 | 5 | `platform` changes meaning on the fetch-all path (M:4497); "Premium" w/o exam = `isPremium` incl. trial (M:4491) vs doc SUBSCRIBED; `whatsappExams*`, entitlement `isTrial`/`accessTier`, payment/order detail never shown |
| SME_TRIAL_EXTENSION_API.md | implemented | 9/10 | 1 | only `entitlements[].isTrial` unrendered; revival rule + notify pairing both live |
| SME_APPLE_IAP_API.md | partial | 7/11 | 4 | `appleOriginalTransactionId` promised, never returned; product ids "from `GET /sme/exams`" is wrong; `notes` detail unused; no subscription-state view (admitted) |
| SME_NOTIFICATIONS_API.md | implemented | 8/10 | 2 | bypass rule not said on send UI; doc omits `params.exam` stamping |
| SME_NOTIFICATION_RULES_API.md | implemented | ~17/22 | 5 | per-language copy not authorable + false "English only today" UI copy (`RuleEditor.tsx:513`); `maxPerUserPerDay` clamp-vs-400; `isLiveNow` ignored |
| SME_NOTIFICATION_RULES_PAGE_GUIDE.md | implemented | 19/22 | 3 | doc errors: `sampleVars` not in `/triggers`; tapped route `/notification-preferences/tapped` wrong; tapRate 0 shown as "not yet available" |
| SME_FEEDBACK_API.md | implemented | 14/21 | 7 | chat table renders every rating as thumbs-down (`'up'` vs `UP`); per-language survey maps + chat `examId`/`updatedAt` invisible; `responseCount` ≠ answers; `GET /sme/content/simulations/:id` doesn't exist |
| SME_OFFERS_API.md | implemented | ~19/22 | 3 | `contentByLanguage` unrendered; doc §4 stale ("form has no priceInPaise…"); doc §7.1 wrong product-id source |
| SME_CAMPAIGN_ABANDONED_API.md | implemented | 11/13 | 2 (doc) | VIEW/TAP rows purged after 30 d / 90 d vs "complete for full history"; `signal` re-labels, doc reads as subset |
| SME_CAMPAIGN_FORM_GUIDE.md | implemented | 12/12 | 0 | all ADD/CHANGE/REMOVE done; both false banners gone |
| SME_CAMPAIGN_ANALYTICS_PAGE_GUIDE.md | implemented | 11/11 | 0 (1 inherited doc error) | internal list lacks `stanzasoft.ai`/`prepmonkey.*` (declined-by-user topic — not proposed) |
| SME_BANNERS_API.md | implemented | 9/11 | 2 (minor) | `?exam=` upstream filter never used (client-side filter); `GET :id` unused |
| SME_EXAMS_API.md | implemented | ~17/21 | 4 | `pyq_open` undocumented, not addable, labelled "legacy key"; `paywallContentByLanguage` unrendered; §5 "broadcast cannot be segmented by exam" false; trial-14 passages stale |

Nothing in this group is "not at all": every doc's core surface is on screen. The gaps cluster in three places: **(1) the 2026-10-30 per-language layer** (rule copy, survey content, campaign `contentByLanguage`, exam `paywallContentByLanguage`) is proxied but has no portal UI at all, and one screen says the opposite; **(2) a few doc claims are wrong about our own code** (below); **(3) the `/sme/users` fetch-all path** re-implements filters with different definitions.

## Contradictions

1. **Apple — `appleOriginalTransactionId`.** SME_APPLE_IAP_API §2.2: "the Apple identity is `appleTransactionId` + `appleOriginalTransactionId` (the stable id that survives renewals)"; §4: "Group by `appleOriginalTransactionId` (visible in detail)". Code: `rg appleOriginalTransactionId backend/src/modules/sme` → 0 hits; `formatOrder` `sme-transaction.service.ts:353–386` never emits it (column exists at `prisma/schema.prisma:564`).
2. **Product ids "from `GET /sme/exams`".** SME_APPLE_IAP_API §0/§5 and SME_OFFERS_API §7.1 ("Source the product id from `GET /sme/exams`"). Code: `sme-exam.service.ts:90–96` returns exam rows only; plans live on `GET /sme/exams/:id/plans` (`sme-exam.controller.ts:103`). The portal already knows (`offers.api.ts:8–9`).
3. **Chat feedback rating casing (portal render bug).** Doc capture: `"rating": "UP"` (SME_FEEDBACK_API §2, measured block); enum `UP|DOWN` `schema.prisma:2239–2242`; service returns raw rows `sme-feedback.service.ts:254–263`. Portal: `m.rating === 'up'` `ChatFeedbackTab.tsx:188` → every row draws a thumbs-down, including under the "up" filter. Type comment `api.ts:3054` says `// up | down`.
4. **Abandonment history.** SME_CAMPAIGN_ABANDONED_API §1: "Every other signal is complete for the full campaign history." Code: VIEW from raw `api_usage` (`sme-offer.service.ts:547`), purged at 30 days (`api-usage-rollup.service.ts:30, 126–127`); TAP from `auth_events` (:522), purged at 90 days (:32). Only ORDER (orders table) is permanent. The analytics page guide's "since 20 Aug" badge inherits the error.
5. **`signal` semantics.** SME_CAMPAIGN_ABANDONED_API §2 lists `order` "(certain only)", `tap` "(inferred only)" and §1 says signals "partition the audience — they never double-count". Code: the label is computed only from the requested CTEs (`sme-offer.service.ts:491–493, 579–583`), so `signal=view` labels an ordering viewer VIEW; separate calls overlap. Portal proxy M:7506–7511 and `OfferAnalytics.tsx:62–66, 326–361` document and work around it.
6. **`maxPerUserPerDay`.** SME_NOTIFICATION_RULES_API §7: "clamped server-side to 0–5. A value outside that is corrected, not obeyed." Same doc's error table + PAGE_GUIDE §6 say the API 400s; DTO `@Max(5)` `notification-rule.dto.ts:293–298` → 400. (Service also clamps, `notification-settings.service.ts:97–99`, but the DTO rejects first.)
7. **Rule copy languages.** SME_NOTIFICATION_RULES_API §2 "Only `en` is populated today" and §9 "Language variants are stored but not populated — every rule is English today" vs the doc's own 2026-10-30 header and `notification-rule.dto.ts:20–29` (per-language map, recipients get their UI language). Portal repeats the stale line as UI copy: `RuleEditor.tsx:513`.
8. **`sampleVars`.** PAGE_GUIDE §2: "The `/triggers` response now also carries `sampleVars`." Code: `listTriggers()` `notification-rule.service.ts:52–67` has no `sampleVars`; it is used only server-side for test sends (`rule-runner.service.ts:447`).
9. **Tap route.** PAGE_GUIDE §5: "`POST /notification-preferences/tapped` stamps `notification_dispatch.tapped_at`". Code: `notification-preferences.controller.ts:59` `@Controller('notifications')` + :97 `@Post('tapped')` → `POST /notifications/tapped` (SME_NOTIFICATIONS_API §4.2b is correct).
10. **Exam-scoped broadcast.** SME_EXAMS_API §5: "`POST /sme/notifications/broadcast` is a topic blast and cannot be segmented by exam." Code: `sme-notification.controller.ts:27–61` takes `exam` → `exam_<slug>` (and SME_PORTAL_API §4.1 / SME_NOTIFICATIONS_API §2.1 document it).
11. **Simulation detail route.** SME_FEEDBACK_API §1 detail table: "Read it from `GET /sme/content/simulations/:id`." No such route on our side (`sme-content-ingest.controller.ts`; AUDIT §2.3); the portal synthesises it from the list (M:8436–8444).
12. **`responseCount`.** SME_FEEDBACK_API §4 "did anyone answer? (`responseCount`)". Code `sme-survey.service.ts:254` = `_count.responses` over every `SurveyResponse.status` (answered + snoozed + dismissed). Portal labels the column correctly ("Reached") but its close dialog says "N responses already collected" `SurveysTab.tsx:181`.
13. **`/sme/users` filters on the portal's fetch-all path.** Doc §1.1 `platform` = active device token (`sme-user.service.ts:119–121`) vs portal `_sme_user_matches` matching `provider`/`subscriptionSource` (M:4497–4500); doc `premium=true` = SUBSCRIBED-and-not-expired vs portal `isPremium` incl. trial (M:4490–4491). Both apply whenever any premium/premiumState/date filter is set.
14. **Grant `examId` default.** SME_PORTAL_API §1.6: examId optional, "Defaults `upsc-cse`". Portal proxy refuses without it (M:4685–4686) — intentional, safer; the doc still advertises the footgun default.
15. **`pyq_open`.** SME_EXAMS_API §8 "Valid keys (anything else is a 400)" lists 10 keys; code accepts an 11th, `pyq_open` (`app-config.service.ts:51, 58–69`), enforced on `/pyq/:id/details` (`pyq.controller.ts:386–403`). Portal labels any registry-unlinked cap "legacy key for older app builds" (`FeaturesTab.tsx:232`), which is false for `pyq_open`.
16. **Stale passages (no code contradiction, just out of date):** SME_OFFERS_API §4 "the portal's offer form has no `priceInPaise` / `appleOfferCode` / `appleProductId` input" (it has all three, `Offers.tsx:666–718`); SME_EXAMS_API §2 example `trialDays: 14`, §9 "upsc-cse was left untouched at 14", §12 "the trial is 14 days from the epoch entry" (upsc-cse is 3 days since 2026-09-25 per the same doc's §4); SME_NOTIFICATIONS_API has no `params.exam` (DEEP_LINKS §6 + `withExamParam`); SME_FEEDBACK_API §2/§7 "no `mentor` key on production… every forward fails" — likely stale (memory says `DIFY_APP_KEYS` set 2026-10-12), UNVERIFIED, read-only SQL to settle it: `SELECT count(*) FILTER (WHERE dify_forwarded) AS forwarded, count(*) AS total FROM chat_message_feedback WHERE created_at >= '2026-10-12';`.

Brief claims checked and **not** contradicted: two exam axes + `entitlements[]` rendering; per-exam revoke dialog + `?exam=all`; `reconcileHint` chips; `includeTest`; trial revival + notify pairing; broadcast/segment `exam` + `exam_<slug>` echo; rules form from `/triggers`, `{{variable}}` chips, preview funnel, Enable vs 0%, `tapped` never beside `isRead`; feedback `?exam=`/`none`, `context.examIds`, `targetExams`, `breakdown=exam`, CSV trailing `exam`; offers `examId` required, `priceInPaise` exposed, `requiresCode`, `appleOfferCode`/`appleProductId`, `bannerImageUrl`, `bonusDays` removed, both "display strings" banners removed (portal `CLAUDE.md:370`); abandoned `minAppVersion`; the analytics redesign (verdict→funnel→chase→bought, one merged table, internal toggle, `/purchased` band); banners tier×platform×age, allow-list, status machine; every §7–§12 exam control except `pyq_open` and the per-language paywall map.

## Processes stopped

None. No background processes, servers, or browsers were started; all work was read-only `rg`/`sed`/`git blame` (git blame with `GIT_PAGER=cat`). No `git fetch/pull/checkout`, no SQL run, no `.env*` read.

Files written: this file only.
