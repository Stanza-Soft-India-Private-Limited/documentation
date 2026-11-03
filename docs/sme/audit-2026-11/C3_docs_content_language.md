# C3 — Docs ledger: content, language, deep links, rollout, index (audit C, agent 3 of 3)

Read-only pass, 2026-11-03. Paths: `B:` = `/Users/yashwanth/Desktop/projects/backend/src/…`, `M:` = `portal/app/main.py`, `U:` = `portal/admin-ui/src/…`, `D:` = `backend/docs/sme/…`.
Verdicts: **RENDERED** (UI sends/shows it) · **PROXIED-ONLY** (main.py forwards it, no UI sends/shows it) · **BACKEND-ONLY** (we have it, the portal never calls it) · **ABSENT** (doc promises, backend lacks) · **CONTRADICTED** (doc and code disagree; both quoted) · **n/a** (app-side, not a portal surface).

**Environment fact that changes several verdicts (probed 2026-11-03, public GETs only, no SQL):**
- `GET https://app.stanzasoft.ai/api/v1/i18n/languages` → **404 "Cannot GET"**; the same on staging → 200 `{languages:[en,hi]}`.
- `GET /api/v1/cms/pyq?limit=1`: prod row has `paperLabel/questionImageUrl/imageCaption/boardDeleted` (10-01 build is live) but `language: "ENGLISH"`; the staging row has `language: "hi"`.
- So **prod still runs a pre-2026-10-23 image**. Everything dated 2026-10-23 and 2026-10-30 (`/sme/i18n/*`, `language` writes, `X-Lang`, `…ByLanguage`, mains `marks`, empty simulations hidden) exists **on staging only**. The portal talks to `MUX_BACKEND_URL` (prod). WORKLOG:37 still lists the prod rollout as pending (user by hand).

Proof of absence. Control `/sme/users`: 59 hits in `portal/app`, 17 in `admin-ui/src`. Zero hits in both trees for each of: `/sme/i18n`, `i18n`, `translations/`, `keys/sync`, `uiLanguage`, `ui_language`, `X-Lang`, `X-Served-Lang`, `servedLang`, `ByLanguage`, `displayNameI18n`, `paperLabel`, `questionImageUrl`, `imageCaption`, `boardDeleted`, `QUESTION_WITHDRAWN`, `pyq-figures`, `pyq/bulk`, `mains/bulk`, `link_events`, `/links/`, `by-source`, `reparse`. The only `"language"` in `portal/app` is `content_push.py:68` (`"language": "ENGLISH"`).

Structural block on `X-Lang`: `_sme_proxy` sends a fixed header dict (`M:4369` `{"x-api-key", "Content-Type"}`), and `cms_proxy._make_request` (`cms_proxy.py:31–39`) takes only `params`/`json_data`. Neither proxy can forward a client header. **Adding X-Lang needs a proxy change, not just a UI change.**

---

## SME_CONTENT_INGEST_API.md (D: 1128 lines, last change 10-30)

| # | Promise (doc §) | Backend | Portal proxy | Portal UI | Verdict |
|---|---|---|---|---|---|
| 1 | `POST /sme/content/pyq` (§3.1) | `sme-content-ingest.controller.ts:193` | M:8253/8259 (body passed whole) | `content.api.ts:152` via `usePYQ.ts:41` | RENDERED |
| 2 | `POST /sme/content/pyq/bulk` 1–100, all-or-nothing (§3.2, §4) | :215 | M:8262/8268 (`_bulk_items_body` M:8222–8241) | none (0 hits for `pyq/bulk`). The Excel import uses keyless `/cms/pyq/bulk` instead (row 15) | PROXIED-ONLY |
| 3 | `PATCH /sme/content/pyq/:id` merge (§3.3) | :251 | M:8271 → `_logged_question_patch` M:8066: a before-read on keyless `GET /cms/pyq/{id}` (M:8078), then `PATCH /sme/content` (M:8098); `_edit` is popped before forwarding | `content.api.ts:171` (diff-only `pick`, :105–117) | RENDERED |
| 4 | `GET /sme/content/pyq` `?exam=…` (§3.4) | :279 | M:8244 (`dict(request.query_params)`) | none. Lists go through `/cms/pyq` (`api.ts:1537`) | PROXIED-ONLY |
| 5 | `POST /sme/content/mains` | :305 | M:8291 | `content.api.ts:152` via `useMains.ts:41` | RENDERED |
| 6 | `POST /sme/content/mains/bulk` | :328 | M:8300 | none (Excel goes to `/cms/mains/import-excel`, `api.ts:1600`) | PROXIED-ONLY |
| 7 | `PATCH /sme/content/mains/:id` | :358 | M:8309 → M:8066 | `content.api.ts:171` | RENDERED |
| 8 | `GET /sme/content/mains` | :383 | M:8282 | none (`api.ts:1578` uses `/cms/mains`) | PROXIED-ONLY |
| 9 | `POST /sme/content/simulations` (§3.6) | :439 | M:8329 | `simulations.api.ts:99` | RENDERED |
| 10 | `PATCH /sme/content/simulations/:id` | :466 | M:8338 | `simulations.api.ts:103` | RENDERED |
| 11 | `GET /sme/content/simulations?exam=`, inactive rows included (§3.7) | :518 | M:8320 | `simulations.api.ts:89–91`; status filter `Simulations.tsx:158` | RENDERED |
| 12 | `POST …/simulations/questions/bulk` 1–500, skip dup `externalId` (§3.8) | :413 | M:8348 | `simulations.api.ts:115`, `SimulationQuestionsDrawer.tsx:148` (`SIM_BULK_MAX = 500`, simulations.api.ts:85) | RENDERED |
| 13 | `POST …/simulations/:id/assign-questions`: replaces the list (§3.9) | :493 | M:8359 | `simulations.api.ts:142` | RENDERED |
| 14 | **`examIds` required on every create; `*` sentinel exempt from the catalogue check** (§2.1) | `requireExamIds` :147–164 (`ALL_EXAMS` skips `assertExists`; `[]`/missing → `EXAM_IDS_REQUIRED_MESSAGE`, `exam.constants.ts:147`) | passthrough | `PYQ_REQUIRED`/`MAINS_REQUIRED` start with `examIds` (`content.api.ts:51–58`); no default (`PYQ.tsx:57`, `Mains.tsx:30`); `ExamIdsPicker.tsx:27–35` offers "All exams" = `*`; `normaliseExamIds` makes `*` absorb the rest (`content.api.ts:94–97`) | RENDERED (single-row create only) |
| 15 | **"Use `/sme/content/*` for every new integration"** (§0). Does the portal's bulk/Excel path send `examIds`? | `/cms/*` falls back: `resolveIngestExamIds` → `['upsc-cse']` (`exam.constants.ts:126–137`; `cms-pyq.service.ts:62–67`) | M:8506 `parse_pyq_from_bulk` and M:8609 `parse_mains_sheets`: the item dicts in `excel_parser.py:174–193` (PYQ) and `:272–281` (Mains, `examName` hard-coded `"UPSC CSE Mains"`) carry **no `examIds`, no `language`**. They are pushed through `import_and_push` (`excel_parser.py:306`) to keyless `/cms/pyq/bulk` (`cms_proxy.py:122`) with no audit row | Honest warning copy: `PYQ.tsx:150` "The Excel importer does not set exams, so every imported row is filed under UPSC CSE." and `Mains.tsx:277` | **Not followed.** Bulk goes through the keyless legacy path, untagged → UPSC by default. The UI says so honestly |
| 16 | 400 `message` is a single string for the exam checks, an array for validators (§2.3, §8) | — | `_sme_proxy` flattens a list to `"; "` (M:4394–4398) | `errText` (`content.api.ts:120–123`) | RENDERED |
| 17 | PATCH: omitted `examIds` untouched; `[]` → 400 (§2.4) | `optionalExamIds` :170–175 | passthrough | diff-only body (`content.api.ts:155–160`); clearing every exam is refused client-side by `missingRequiredChanged` (:83–92) | RENDERED |
| 18 | Every write audited `CONTENT_INGEST`; no read API (§2.5) | `recordIngest` :181 | the portal keeps its OWN per-person edit log in DynamoDB (M:8045–8100, `/content-edits`) | ContentEdits.tsx | BACKEND-ONLY (by doc design) |
| 19 | `isVerified`/`isActive`/`repeatFrequency` not settable (§3.3 correction) | `UpdatePYQPaperDto` has 0 `isVerified`/`isActive` (generated `update-pyqpaper.dto.ts`); `forbidNonWhitelisted` at `main.ts:249` | — | shown read-only, never sent (`content.api.ts:9–12`; `QuestionFlags` `PYQ.tsx:764`) | RENDERED (correctly withheld) |
| 20 | `tags`/`keywords`/`relatedTopics` required, non-empty (§3.1, §3.5) | generated DTO | — | `content.api.ts:51–58`; `CommaListInput required` `Mains.tsx:592` | RENDERED |
| 21 | **`marks` 1–50 on mains create, bulk and PATCH** (§3.5, 10-30) | `MainsMarksDto` `sme-content-ingest.dto.ts:115–130`; `SmeCreateMainsDto`/`SmeUpdateMainsDto` :133–142 (**staging only**) | passthrough (M:8291) | `MAINS_CONTENT_FIELDS` omits `marks` (`content.api.ts:44–48`). Comment :43 says "`marks` is NOT settable (400 if present)"; `Mains.tsx:30–31` and the hint at `:597` say "the marks field itself cannot be set through the API" | BACKEND-ONLY. The portal copy is **true on prod today** and becomes false when 10-30 ships |
| 22 | `language` on writes, STRICT on this surface (§3.1 note, LANGUAGE §1.1) | `requireLanguage`/`optionalLanguage` :92–115 (staging only) | the proxy would forward it (dict body) | 0 hits; not in `PYQ_CONTENT_FIELDS`/`MAINS_CONTENT_FIELDS` | BACKEND-ONLY |
| 23 | `X-Lang` exact filter on the three lists | :117–125, Swagger :287 | **cannot be forwarded**: fixed headers at M:4369 | 0 | BACKEND-ONLY (the proxy structurally drops it) |
| 24 | **App's `GET /simulations` hides an active simulation with 0 questions** (§3.6, 10-30) | `simulation.service.ts:35–46` `questions: { some: {} }` (staging only) | — | `Simulations.tsx:96` "Active with no questions. Learners may see an empty test." | CONTRADICTED vs staging code (still true on prod until rollout) |
| 25 | Assign replaces; a shorter list unassigns (§3.9) | :493 | M:8359 (M:8374–8383 comment: no endpoint returns the currently assigned ids) | Drawer warns it is a replace | RENDERED |
| 26 | Verify a load with `content-counts` → `exclusive` (§6) | sme-exam controller | M:7252/7259 | `ContentCountsTab.tsx:17–26`, `ExamList.tsx:42,99` | RENDERED |
| 27 | §7 not mirrored: DELETE, GET-by-id, `externalId` lookup, psychometric → keep `/cms` | `/cms/*` controllers | M:8558/8661 (delete), M:8533/8636 (get), M:8409→8401 (lookup), M:8451→8459 (sim soft delete), M:8683–8833 (psychometric) | `api.ts:1541–1666`, `simulations.api.ts:111,132` | RENDERED (keyless, as the doc says) |
| 28 | **PYQ `paperLabel`** (§9.1) | `create-pyqpaper.dto.ts:140`, `update-pyqpaper.dto.ts:147` (live on prod) | passthrough | 0 hits; not in `PYQ_CONTENT_FIELDS` (`content.api.ts:36–41`), so even a new form field would be dropped by `pick()` | BACKEND-ONLY |
| 29 | **`questionImageUrl`** | :148 / :155 | passthrough | 0 | BACKEND-ONLY |
| 30 | **`imageCaption`** | :156 / :163 | passthrough | 0 | BACKEND-ONLY |
| 31 | **`boardDeleted`**: withdraw a question (§9.1, §9.3) | :164 / :171; rule in `pyq-withdrawn.util.ts:89–113` | passthrough | 0. **The portal cannot withdraw a question** | BACKEND-ONLY |
| 32 | Answer-key rule: one uppercase `A`–`E`; null only if `boardDeleted` (§9.2) | `pyq-withdrawn.util.ts:65` `/^[A-E]$/`, :113 (`''`/null refused unless `boardDeleted`) | Excel: `correctAnswer` sent raw via `_safe_str` (`excel_parser.py:184`), not upper-cased, so a lowercase or blank cell 400s its **whole 100-row batch** (code-proven, not run) | Editor select offers only A–D, and displays `'A'` when the row's answer is null (`PYQ.tsx:639–646`). No `optionE` input (`:622`), although `optionE` is in `PYQ_CONTENT_FIELDS` | RENDERED (partial): E not settable; a withdrawn row looks answered "A" |
| 33 | `409 QUESTION_WITHDRAWN` (§9.3) | `pyq-withdrawn.util.ts:19–33`, guard `pyq-withdrawn.guard.ts` | — | — | n/a (app routes only; the doc says "nothing on /sme returns them") |
| 34 | `content-counts` prelims row gains `boardDeleted` (§9.3) | exam service | passthrough M:7252 | `exams.api.ts` type has no `boardDeleted` (0 hits) | PROXIED-ONLY |
| 35 | Filter-config prelims `questionCount` = playable rows only (§9.3) | filter-config service | passthrough | rendered unlabelled as "question row(s)" (`FilterConfig.tsx:293,538`) | RENDERED (meaning unlabelled) |
| 36 | **Figure upload `POST /sme/media/upload-url` `folder:"pyq-figures"`** (§9.4) | `sme-media.dto.ts:8` | M:7813 forwards `filename, contentType, folder` | the folder type is `'offers' \| 'banners'` (`api.ts:4213,4228`); `exams.api.ts:209` uses `exam-logos`; 0 hits for `pyq-figures` | PROXIED-ONLY |

**Per-doc verdict: partial, about 17 of 36 rows RENDERED.** Most consequential missing items:
1. **The four PYQ fields are unauthorable in the portal** (rows 28–31, 36). APPSC figures and paper labels can be loaded only by script, and a board-withdrawn question cannot be withdrawn from the portal. `PYQ_CONTENT_FIELDS` would strip them even if a form sent them.
2. **The Excel bulk path is keyless and untagged** (row 15): `/cms/*` with no `examIds` falls back to `upsc-cse`, with no audit row, and a single lowercase answer letter 400s a whole batch of 100. The UI copy is honest about the exam.
3. **No `language` and no `X-Lang` anywhere** (rows 22–23). The proxy cannot carry the header.

**Stale portal copy:** `content.api.ts:43`, `Mains.tsx:30–31`, `Mains.tsx:597` (marks) and `Simulations.tsx:96` (empty test) are each correct on prod and wrong on the staging image; flip them in the same release.
**Doc's own stale passages:** the "Changed 10-30" header lines describe staging-only behaviour without saying so (unlike LANGUAGE:33–35). §3.1 still embeds the 2026-09-21 `"ENGLISH"` capture with a note, which is fine.

---

## CMS_ADMIN_API.md (D: 1284 lines, legacy `/cms/*`)

| # | Promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `GET /cms/pyq` (page, limit, exam, subject, year, difficulty, search, isActive, isVerified) | `cms-pyq.controller.ts:55–79`, `CmsPyqFiltersDto` | M:8464–8489 forwards all 9 | `api.ts:1527–1537`; `?exam=` `PYQ.tsx:199,218`; year, difficulty and isVerified in `CmsFilterBar.tsx:54,84,112`; isActive has no control | RENDERED (`isActive` PROXIED-ONLY) |
| 2 | `GET /cms/pyq/:id` | :83 | M:8533 | `api.ts:1541` (edit deep link `?edit=`) | RENDERED |
| 3 | `POST /cms/pyq` | :41 | M:8521 | `createPYQ` `api.ts:1545`, never called | PROXIED-ONLY |
| 4 | `POST /cms/pyq/bulk` | :48 | M:8494 (bare list, wrapped to `{items}` at `cms_proxy.py:122`) | only via Excel import M:8506 (`api.ts:1559`) | RENDERED (via Excel) |
| 5 | `PATCH /cms/pyq/:id` | :92 | M:8545 | `updatePYQ` `api.ts:1549` never called (PATCH goes to `/sme/content`) | PROXIED-ONLY |
| 6 | `DELETE /cms/pyq/:id` (hard) | :104 | M:8558 | `api.ts:1553`, `PYQ.tsx:505` | RENDERED |
| 7–12 | Mains: same six | `cms-mains.controller.ts:41–104` | M:8572–8668 | list `api.ts:1578`, get :1582, delete :1594, bulk via Excel :1600; create/patch dead (:1586, :1590) | 4 RENDERED, 2 PROXIED-ONLY |
| 13 | `/cms/simulations` GET/POST/PATCH | `cms-simulation.controller.ts:33,52,59` | none (the portal uses `/sme/content`) | — | BACKEND-ONLY (superseded, correct) |
| 14 | `DELETE /cms/simulations/:id` soft | :69 | M:8451 → M:8459 | `simulations.api.ts:111` | RENDERED |
| 15 | `GET /cms/simulations/questions?externalIds` | :76 | M:8409 → M:8401 (≤500 ids) | `simulations.api.ts:132` | RENDERED |
| 16 | `/cms/simulations/questions/bulk`, `/:id/assign-questions` | :94, :107 | none (keyed mirror used) | — | BACKEND-ONLY (superseded) |
| 17 | Psychometric questions: list/get/create/patch/delete/bulk | `cms-psychometric.controller.ts:45–98` | M:8683–8757 | list/create/update/delete `usePsychometric.ts:33,52,71`; get dead (`getPsychometricQuestion`); bulk has no UI caller | 4 RENDERED, 2 PROXIED-ONLY |
| 18 | Test sets: list (page, limit only), get, create, patch, delete | :108–144 | M:8771–8826. **The list forwards `isActive`, `search`** (M:8781), which the backend silently ignores (`@Query('page')`, `@Query('limit')` only, :118–120) | `useTestSets` sends page/limit only (`Psychometric.tsx:524`); get dead (`getTestSet`) | 4 RENDERED, 1 PROXIED-ONLY |
| 19 | `examIds` optional → `['upsc-cse']` + WARN, never 400 (§0) | `exam.constants.ts:126`; `cms-pyq.service.ts:62–67` | — | warned in UI (`PYQ.tsx:150`, `Mains.tsx:277`) | RENDERED (honestly disclosed) |
| 20 | §0 migration map: new integrations on `/sme/content/*` | — | both exist | single-row create/patch keyed; bulk, lists, get, delete keyless | partial |
| 21 | `language` LENIENT on /cms (10-30) | `cms-pyq.service.ts:37` (staging only) | — | never sent (Excel included) | BACKEND-ONLY |
| 22 | **PATCH example `{"difficulty":"HARD","isVerified": true}`** (D:393–396, mains D:483–486) and workflow step 4 "PATCH … to set `isVerified: true`" (D:1202); create examples carry `"isVerified": false` (D:148, D:189); field tables mark `isVerified` writable (D:829, D:860) | generated `create-pyqpaper.dto.ts` and `update-pyqpaper.dto.ts` have **0** `isVerified`/`isActive` (same for `mains-paper`); `forbidNonWhitelisted: true` (`main.ts:249`) → `400 property isVerified should not exist` | — | the portal already knows (`content.api.ts:9–11`) | **CONTRADICTED.** PYQ and Mains only; psychometric `isVerified` IS writable (`psychometric/dto/create-question.dto.ts:82`), so D:224 and D:720 are fine |

**Per-doc verdict: partial, about 18 of 30 route/param rows RENDERED** (the gaps are mostly correctly superseded routes or dead UI functions). Most consequential:
1. The Quick Start and field reference teach `isVerified` on PYQ/Mains create/PATCH, which 400s the whole request (row 22). Copying the Quick Start fails.
2. Psychometric has no keyed mirror at all (AUDIT §2.2), so `/psychometric` is 100% keyless.
3. The test-set list proxy forwards `isActive`/`search` that we ignore. Harmless today, but a future UI filter would silently do nothing.

---

## CONTENT_DOC_SME_API.md (D: 356 lines, `/content-doc-admin`)

| # | Promise | Backend | Proxy / service | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `POST /content-doc-admin` (title, subject, topic, rawMarkdown; status, language, sourceId, examIds) | `content-doc-admin.controller.ts:35` | `content_push.py:61–78`: sends `examIds` from `exam_ids` (default `['upsc-cse']`, :17–22), `"status": "PUBLISHED"` hard-coded (:67), `"language": "ENGLISH"` (:68, accepted leniently as `en`), `sourceId` = portal item id (:69) | `ContentEditor.tsx:302,308` | RENDERED |
| 2 | `PATCH /content-doc-admin/:id` (re-sync) | :94 | `update_content` `content_push.py:106–123`: **always** sends `rawMarkdown`, `examIds` and **`"status": "PUBLISHED"`** (:112) | "Re-sync … will update the existing remote document." (`ContentEditor.tsx:303`) | RENDERED, with two unstated side effects (rows 3–4) |
| 3 | Doc: "If `rawMarkdown` is included: all existing questions are deleted and recreated" | `content-doc.service.ts:334–339`: only **when the markdown differs** (`dto.rawMarkdown !== existing.rawMarkdown`). `UserQuestionAttempt.question … onDelete: Cascade` (`prisma/schema.prisma` ~1109) | — | Re-sync confirm says nothing about it | **CONTRADICTED (minor):** the code is narrower than the doc. **Omitted by both doc and UI:** a re-sync with edited markdown deletes every learner's quiz attempts on that document (cascade) |
| 4 | Status lifecycle DRAFT → review → PUBLISHED → ARCHIVED (Workflow §1–6) | — | First push creates as PUBLISHED (no draft review). A re-sync of an ARCHIVED doc **re-publishes it** (:112), and the PATCH branch does not update the stored `remote_status` (M:2687–2697, only the POST branch sets it at M:2716), so the dropdown can show ARCHIVED while the doc is live | `ContentEditor.tsx:567–573` status select; M:2732 → `change_remote_status` (`content_push.py:145`) | RENDERED (status values DRAFT/PUBLISHED/ARCHIVED match). Re-sync silently un-archives |
| 5 | `examIds` optional, default `['upsc-cse']`, `*` sentinel, read back the normalised value | `resolveIngestExamIds` (`content-doc.service.ts:90,322,393,422`) | `_stored_exam_ids` (`content_push.py:25–32`), persisted as `remote_exam_ids` | `ContentExamPanel.tsx:19–35` shows the stored tags and flags a mismatch | RENDERED |
| 6 | `GET /content-doc-admin` list `?exam=`, status, search, X-Lang | :43, X-Lang :60 | none | — | BACKEND-ONLY |
| 7 | `GET /content-doc-admin/by-source/:sourceId` (idempotent ingest) | :74 | not used: push decides POST vs PATCH from its own `remote_doc_id` and re-POSTs on a PATCH 404 (M:2681–2708) | — | BACKEND-ONLY |
| 8 | `GET /content-doc-admin/:id` | :84 | M:6611–6625 (feedback report target only) | `ReportTargetCard` | RENDERED (feedback only) |
| 9 | `POST /:id/reparse`, `POST /reparse-all` | :112, :104 | none | — | BACKEND-ONLY |
| 10 | `DELETE /content-doc-admin/:id` soft | :122 | none | — | BACKEND-ONLY |
| 11 | `language` code + X-Lang (10-30 header) | lenient on write (staging) | push sends `"ENGLISH"` (accepted) | — | PROXIED-ONLY (hard-coded) |

**Per-doc verdict: partial, 5 of 11 RENDERED.** Most consequential:
1. **A re-sync with changed markdown cascade-deletes learners' quiz attempts.** The doc says "questions deleted and recreated" but never mentions user data, and the UI confirm says nothing.
2. **A re-sync forces `PUBLISHED`**, un-archiving an archived document, and the portal's `remote_status` goes stale.
3. `by-source`, `reparse`, `reparse-all` and `DELETE` have no portal surface. They are script tools, so this is acceptable, but nothing in the portal can remove a pushed document.

---

## SME_FILTER_CONFIG_API.md (D: 190 lines)

| # | Promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `GET /sme/filter-config/:examType?exam=` (validIconKeys, entries, effective/override, questionCount, hidden included) | `sme-filter-config.controller.ts:57–72` (strict `assertExists`, :53) | M:6488/6495, `_filter_config_params` M:6483 | `api.ts:2918`; `FilterConfig.tsx:364–377` (`?exam=` URL param; a 400 is surfaced) | RENDERED |
| 2 | `PUT …/:examType/:subject?exam=`: displayName / iconKey / hidden / sortOrder, null = clear | :75–94 | M:6498/6508 (allow-list of those 4 keys) | `FilterConfig.tsx:72–112` three-state set/clear/unset; `api.ts:2927` | RENDERED |
| 3 | Icon keys only from `validIconKeys` | service | passthrough | `FilterConfig.tsx:211` select from `validIconKeys` | RENDERED |
| 4 | `POST …/mains/:subject/optional?exam=` `{isOptional}`; flips `*`-shared rows for every exam | :105–122 | M:6536/6549 (allow-list `isOptional`) | `FilterConfig.tsx:261–323` confirm, with a shared-row sample via portal-built M:6514 (→ keyless `/cms/mains`, first 100 rows, `complete` flag) | RENDERED |
| 5 | Flip keeps the raw name (no auto-strip) | — | — | `FilterConfig.tsx:299` "does not auto-rename" | RENDERED |
| 6 | **`displayName` as a per-language map; `effective.displayNameByLanguage`; `override.displayNameI18n`** (10-30 header; LANGUAGE §3.2) | `sme-filter-config.service.ts` (staging) | the PUT allow-list would pass a map through as `displayName` | 0 hits for `ByLanguage`/`displayNameI18n`. `OverrideChips` keys on `override.displayName != null` (`FilterConfig.tsx:344`), so a Hindi-only override renders as "—" (no override). "Clear" sends `displayName:null` = English only | BACKEND-ONLY |
| 7 | Doc body examples (§1, §2) | — | — | — | Stale: they omit `displayNameByLanguage` and `displayNameI18n`, which the 10-30 header says are on every response |

**Per-doc verdict: implemented for the pre-10-30 contract (5 of 5 core rows RENDERED); the language-map row is BACKEND-ONLY.** Missing: a second-language display name (row 6). Doc stale: §1–§2 JSON examples predate the language map.

---

## SME_LOCALIZATION_API.md (D: 1342 lines, `/sme/i18n/*`, 2026-10-23)

| # | Route / promise | Backend `sme-i18n.controller.ts` | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `GET /sme/i18n/languages` (coverage, draft state) | :47 | 0 | 0 | BACKEND-ONLY |
| 2 | `POST /sme/i18n/languages` | :63 | 0 | 0 | BACKEND-ONLY |
| 3 | `PATCH /sme/i18n/languages/:code` | :81 | 0 | 0 | BACKEND-ONLY |
| 4 | `POST …/:code/publish` | :106 | 0 | 0 | BACKEND-ONLY |
| 5 | `GET …/:code/preview` | :129 | 0 | 0 | BACKEND-ONLY |
| 6 | `GET /sme/i18n/labels` | :145 | 0 | 0 | BACKEND-ONLY |
| 7 | `PATCH /sme/i18n/labels/:key` (English, `sourceEditedBySme`) | :169 | 0 | 0 | BACKEND-ONLY |
| 8 | `GET /sme/i18n/translations/:code` | :191 | 0 | 0 | BACKEND-ONLY |
| 9 | `PUT /sme/i18n/translations/:code` (all-or-nothing) | :217 | 0 | 0 | BACKEND-ONLY |
| 10 | `DELETE …/translations/:code/:key` | :242 | 0 | 0 | BACKEND-ONLY |
| 11 | `GET /sme/i18n/export/:code[?fill]` | :267 | 0 | 0 | BACKEND-ONLY |
| 12 | `POST /sme/i18n/keys/sync` (release tooling, doc says "NOT the portal") | :287 | 0 | 0 | BACKEND-ONLY (by design) |
| 13 | `uiLanguage` on `/sme/users` rows and detail (§5) | `sme-user.service.ts:137,254` | passthrough M:4574 / M:4641 | 0 hits | PROXIED-ONLY |
| 14 | Brief: "work-queue" localization screens (missing / needs review / unpublished) | — | — | none | ABSENT in the portal |
| 15 | Public `GET /i18n/languages`, `/i18n/bundle/:code` | app routes | — | — | n/a. **Prod 404, staging 200** (probe above) |

**Per-doc verdict: not at all (0 of 12 SME routes reach the portal; proven with the control above).** Most consequential: there is no screen to translate, review or publish UI strings, so every Hindi label change needs a developer script. The surface is also staging-only.

---

## SME_CONTENT_LANGUAGE_API.md (D: 942 lines, 2026-10-30)

| # | Promise | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | Language control = ONE dropdown fed by `GET /sme/i18n/languages` on every content form (brief, D:29–31) | route exists | 0 | 0 | ABSENT in the portal |
| 2 | `language` STRICT on `/sme/content/*` writes (§1.1) | ingest controller :92–115 | passthrough | 0 | BACKEND-ONLY |
| 3 | `language` LENIENT on `/cms/*`, content-doc, `PUT /reels/bulk` (§1.2) | `cms-pyq.service.ts:37`; reels `reels.service.ts:417–457` | Excel items have no `language`; push hard-codes `"ENGLISH"` (`content_push.py:68`); reels bulk body built in `Reels.tsx:1135–1144` has no `language`/`examIds` | 0 | PROXIED-ONLY (push only, as a constant) |
| 4 | `X-Lang` on the four SME lists (§2) | ingest :117–125; content-doc :60 | **cannot be carried** (M:4369; `cms_proxy.py:31–39`) | 0 | BACKEND-ONLY. Once prod rolls, every portal list shows all languages mixed with no language column |
| 5 | Every SME row carries `language` | yes (staging) | passthrough | not rendered (0 hits) | PROXIED-ONLY |
| 6 | §3.1 `paywallContentByLanguage` on exams | (SME_EXAMS) | passthrough | 0 | PROXIED-ONLY |
| 7 | §3.2 filter-config `displayNameByLanguage` | see FILTER_CONFIG row 6 | passthrough | 0 | PROXIED-ONLY |
| 8 | §3.3 campaign `contentByLanguage` / `content` map | (SME_OFFERS) | passthrough | 0 | PROXIED-ONLY |
| 9 | §3.4 survey `titleByLanguage` / `descriptionByLanguage` / `questionsByLanguage` | (SME_FEEDBACK) | passthrough | 0 | PROXIED-ONLY |
| 10 | §3.5 notification-rule `copy` as a map | `notification-rule.dto.ts:120,191` `@IsLanguageMapOf(RuleCopyDto)` | passthrough | `RULE_LANG='en'`, writes `{en:{title,body}}` (`api.ts:2328,2384–2386`); `RuleEditor.tsx:513` "English only today. Other languages are stored but not populated" | RENDERED (English key only; the `{en:…}` map is valid). A Hindi translation can't be edited |
| 11 | §4 hand-edited config (examPhases.copyI18n, daily quotes, task titles) | SQL / code | — | — | n/a (no SME writer, by doc) |
| 12 | D:94 "No reel list filters by language (`GET /reels`, `GET /sme/blog`)" | controller is `@Controller('sme/blogs')` (`sme-blog.controller.ts:30`) | M:6244 `/sme/blogs` | `api.ts:2812` | **CONTRADICTED (typo):** `/sme/blog` → `/sme/blogs` |
| 13 | D:33–35 "Production does not have `i18n_languages` yet … staging only" | — | — | — | TRUE as of 2026-11-03 (prod `/i18n/languages` 404; prod row `language:"ENGLISH"`) |

**Per-doc verdict: not at all on the language axis (1 row RENDERED, English-only; 0 rows expose a second language).** Most consequential:
1. **No `…ByLanguage` field is shown or edited on any form** (exams paywall, offers, filter-config, surveys, rules). A Hindi translation can be stored only via raw API.
2. **No language control on any content form**, and the Excel/import paths never tag, so the portal can only ever create `en` rows.
3. **`X-Lang` is impossible through today's proxy.** After rollout, SME lists mix languages silently, and `content-counts` stays all-language.

---

## DEEP_LINKS.md (D: 257 lines)

| # | Promise | Code | Portal | Verdict |
|---|---|---|---|---|
| 1 | Publish external links on `go.prepmonkey.com` with `src=` (§1, §5 checklist) | app/web | `Offers.tsx:85` `SHARE_LINK_BASE = 'https://app.prepmonkey.com/open/offer'`, used by `CopyLinkButton` (:263) and the editor preview (:1132). No `go.` host, no `src` | CONTRADICTED with the doc's guidance (the campaign share link is the one link the portal builds for marketing) |
| 2 | `?exam=` on every APPSC link (§3.1). On 2.0 an absent `exam` keeps the current exam | app `DeepLinkTarget.kt` | Offers share link has no `exam=` even though every campaign has an `examId`; `BannerCard.tsx:250` shows `https://go.prepmonkey.com/open/offer/{code}` with no `exam` | ABSENT in the portal-built links |
| 3 | `src` / `cid`: never write `cid` | — | not offered anywhere (0 `link_events`, 0 `/links/`) | ABSENT (`src` never offered) |
| 4 | Banner `destinationUrl` is parsed as an in-app link; `exam` ignored (§3.3) | banner allow-list | `banners.api.ts:127` allows `app.`/`go.` `/open/…` only; `BannerEditor.tsx:439–451` | RENDERED |
| 5 | **Route keys (§4): `premium` is the paywall; unknown keys open Home** | `DeepLinkMapper.ROUTES[key] ?: DeepLinkTarget.Home` (`upsc_app/…/DeepLinkTarget.kt:463`); legacy aliases (:476–486) have **no `paywall`** | `BannerEditor.tsx:445–447` quick-fill button sets `https://app.prepmonkey.com/open/paywall` | **CONTRADICTED. A banner made with that preset opens Home, not the paywall** |
| 6 | Push `type` keys = §4 table | — | `NotificationComposer.tsx` offers 40 keys matching §4 exactly (`premium`, `offer`, …) | RENDERED |
| 7 | Push `params.exam` stamped by the backend (§6) | `notification-exam.util.ts` | composer sends no exam param (correct: server stamps it) | RENDERED (by omission) |
| 8 | §7 click exam analytics (`link_events.exam_id`, "since 2026-10-01" label) | `links.service.ts` | no portal screen | BACKEND-ONLY |
| 9 | §5 "Ids come from the SME portal (questions: `GET /sme/content/pyq?exam=…`)" | — | the portal never calls `GET /sme/content/pyq` (row 4 of INGEST); PYQ rows show no copyable id | CONTRADICTED (the id source named does not exist in the portal) |

**Per-doc verdict: partial, 3 of 9 RENDERED.** Most consequential:
1. **The `/open/paywall` banner preset opens Home** (row 5).
2. **The campaign share link has no `exam`** (on 2.0 it opens in the user's current exam, so an APPSC campaign link shared to a UPSC user lands in UPSC), uses the `app.` host (no click is recorded and the iOS hand-off fails from `app.`), and carries no `src`.
3. No click/attribution view exists.

---

## PYQ_MOBILE_CONTRACT_2026-10.md (D: 456 lines, app read contract)

| # | Promise | Backend | Portal | Verdict |
|---|---|---|---|---|
| 1 | `paperLabel`, `questionImageUrl`, `imageCaption`, `boardDeleted` on list/details/mock | `pyq.mobile-contract.spec.ts` guards it | the portal can't author them (INGEST rows 28–31) or show them (0 hits, incl. `ReportTargetCard`) | n/a for routes. The portal is the missing authoring half |
| 2 | Board-deleted → shown, inert, `409 QUESTION_WITHDRAWN` | `pyq-withdrawn.guard.ts` | the portal cannot set `boardDeleted`; a feedback report on a withdrawn question opens an editor whose answer shows "A" (`PYQ.tsx:639`) | n/a / gap |
| 3 | `language` code (10-30) | staging | — | n/a |
| 4 | Indexed in README? | — | **No row in README:51–81**; reached only via `SME_CONTENT_INGEST_API.md` §9 | CONTRADICTED with AUDIT §2.3 "not in README index" (confirmed) |

**Per-doc verdict: n/a (app contract); 0 portal rows.** It matters to the portal only as the spec for the 4 unauthorable fields.

---

## rollout-and-verify.md §7–§9 (portal visibility of each smoke)

| § | Smoke | Portal surface that would show it | Visible? |
|---|---|---|---|
| 7.1 | Bare revoke → 400 "exam is required" | `revokeSmePremium(id, exam)` always sends `?exam=` (`api.ts:2139–2140`) | Yes (the portal never makes the bare call) |
| 7.1 | Offer create without `examId` → 400 | Offers form carries `examId` (`Offers.tsx:190–193` plan lookup by exam) | Yes |
| 7.1 | `?exam=nope` → 400 on users/orders/offers/banners/incidents | ExamSelector only offers catalogue slugs; a 400 surfaces as an error | Partly (cannot be typed in the UI) |
| 7.2 | Per-exam grant/revoke leaves the other exam LIVE | `SnapshotHeader.tsx` renders `entitlements[]`/`activeExamId`/`trialExamId` | Yes |
| 7.3 | Two-axis user filter (`exam` + `premium`) | `AppUsers.tsx`; `premium` triggers the M:4561 full scan + local filter (AUDIT bug class 3) | Yes (via scan) |
| 7.4 | `UNRESOLVED_EXAM`, `reconcileHint`, Apple money fields | `Transactions.tsx` renders `reconcileHint`, `UNRESOLVED_EXAM`, `storefrontCurrency` | Yes |
| 7.5 | Snapshot `quota.examId` | `SnapshotHeader.tsx` (`quota.examId`, 3 hits) | Yes |
| 7.6 | Offers/banners `?exam=` | `Offers.tsx:2040`, Banners page | Yes |
| 8.1 | Migrations via `information_schema` | — | No (DB only) |
| 8.2 | Broadcast topic echo / segment `matchedUsers` / bad slug | `AppUsers.tsx` (matchedUsers) | Yes |
| 8.2 | Survey `targetExams` (`*` refused), `breakdown=exam`, `examNullCount`, CSV `exam` column | `SurveyBuilderModal.tsx`, `SurveyResultsView.tsx` | Yes |
| 8.2 | Feedback `?exam=none`, `examId` on rows | `ReportsTab.tsx`, `ExamSelector` "none" option (`ExamSelector.tsx:13`) | Yes |
| 8.2 | `api_usage_daily` duplicate check | — | No (SQL) |
| 9.1–9.4 | i18n migration, bundle/ETag smoke, `/sme/i18n` 401, sync → publish → enable | none | **No** (0 portal surface) |
| (missing) | **Content-language rollout `20261030000000_content_lang`** (WORKLOG:37) | — | **ABSENT from this doc**: 0 hits for `content_lang`/`20261030`. The 10-30 release has no checklist section here |

**Verdict:** §7 and §8 smokes are largely portal-observable (about 11 of 14); §9 has none. Missing: a §10 for the content-language migration and its probes.

---

## README.md (index, D: 106 lines)

| Line | Claim | Evidence | Verdict |
|---|---|---|---|
| 3 | content-language: strict/lenient, X-Lang, `…ByLanguage`. "Staging only until production has `i18n_languages`." | prod `/i18n/languages` 404; prod row `language:"ENGLISH"` | TRUE |
| 5 | `/sme/i18n/*`; `uiLanguage` on `/sme/users`; rollout §9 | `sme-user.service.ts:137,254`; §9 exists | TRUE (backend); portal 0 |
| 7 | DEEP_LINKS + INGEST §9 | both exist | TRUE |
| 19–23 | "Start here: WHAT_CHANGED_2026-09-21" | no cover note for 10-01, 10-23 or 10-30 | STALE |
| 25–27, 54 | archive/WHAT_CHANGED_2026-07-23 is archived | archive header D:archive:3 "ARCHIVED 2026-09-21 — superseded", its only link `../WHAT_CHANGED_2026-09-21.md` resolves; the other doc names are plain text and all exist | TRUE |
| 57 | Filter-config row: `?exam=` defaults to `upsc-cse` | controller :45–54 | TRUE. Silent on the 10-30 language map |
| 68 | **SME_INSIGHTS row: question-quality "⚠️ still not exam-scoped"** | `sme-content.controller.ts:28–30` takes `?exam=` (since `bfae389`). **And the portal proxy DROPS it:** M:5880–5905 forwards pool…limit, no `exam` | **CONTRADICTED** (backend). The portal half really is unscoped |
| 75 | INGEST row | omits the 10-30 `language`/`marks`/empty-sim changes | STALE (minor) |
| 77 | LANGUAGE row | content is accurate; inherits the `GET /sme/blog` typo (LANGUAGE:94) | TRUE (typo downstream) |
| 79 | CMS row "kept working, not extended" | true; but its examples teach `isVerified` writes that 400 | TRUE with a contradicted body |
| 80 | CONTENT_DOC row "no keyed mirror" | `content-doc-admin.controller.ts` every route `@Public()` | TRUE |
| 81 | rollout row: §7/§8/§9 | no content-language (10-30) section | STALE (incomplete) |
| — | **`PYQ_MOBILE_CONTRACT_2026-10.md` not indexed** | absent from the table D:51–81 | CONFIRMED |
| — | **`pyq_open` undocumented** | 0 hits in `docs/sme/*` except the audit; only `docs/API.md:834–858`. Commit `3dd636e` | CONFIRMED. Portal impact: `FeaturesTab.tsx:85–91` builds cap keys from effective caps + registry `quota`s, and `feature-registry.ts` has no `pyq_open` quota, so **the portal cannot add a `pyq_open` cap to an exam that does not already have one** |
| 84, 92–94 | `../USER_DATA_EXPORT.md`, the superpowers spec | both files exist | TRUE |

**Per-doc verdict: index mostly accurate; 1 contradicted claim (L68), 3 stale (L19–23 cover note, L75, L81), 1 unindexed doc, 1 undocumented feature.**

---

## Reels (asked in the brief; documented only in INGEST §7 and LANGUAGE §1.2)

| # | Item | Backend | Proxy | UI | Verdict |
|---|---|---|---|---|---|
| 1 | `PUT /reels/bulk` fields (`id,title,description,date,assetId,playbackId,status,subject,context,examIds,language`) | `reels/dto/bulk-update-reels.dto.ts:19–123` | M:4152–4171: `BulkUpdateRequest` = `List[Dict]` (`models.py:530`), forwarded whole; `date` normalised to ISO | `Reels.tsx:1135–1144` sends id, title, description, subject, context, date, assetId, playbackId. **No `examIds`, no `language`, no `status`** | `examIds`/`language` PROXIED-ONLY. New reels default to `['*']` (`reels.service.ts:465` `resolveIngestExamIds(…, [ALL_EXAMS])`), so every portal reel is shared by every exam |
| 2 | `DELETE /reels` (every reel), keyless | `reels.controller.ts` | M:4226/4236, `require_admin` | `Reels.tsx:599–612` "Delete All", shown only when `user.role === 'admin'` (:1021) | RENDERED (admin-only in the UI; **keyless upstream**, so anyone who can reach our API can wipe reels) |
| 3 | `/sme/blogs` save `{reelId,title,rawMarkdown}` / PATCH `{title,rawMarkdown}` | `sme-blog.dto.ts:41–70`; parser `reel-blog-parser.service.ts:11–12` (content-doc markdown parser, quizzes stripped) | M:6265–6292 allow-lists exactly those keys | `api.ts:2819–2836` | RENDERED (shape matches) |

---

## Summary

| Doc | Verdict | Rows rendered / total | Missing (top) |
|---|---|---|---|
| SME_CONTENT_INGEST_API.md | partial | ~17 / 36 | 4 PYQ fields + `pyq-figures` unauthorable; Excel bulk keyless and untagged (→ UPSC); no `language`/`X-Lang`; `marks` (staging) |
| CMS_ADMIN_API.md | partial | ~18 / 30 | doc teaches `isVerified` writes that 400; psychometric fully keyless; test-set list forwards ignored params |
| CONTENT_DOC_SME_API.md | partial | 5 / 11 | re-sync cascade-deletes learners' quiz attempts; re-sync forces PUBLISHED and `remote_status` goes stale; no list/delete/reparse UI |
| SME_FILTER_CONFIG_API.md | implemented (pre-10-30) | 5 / 6 | per-language `displayName` |
| SME_LOCALIZATION_API.md | not at all | 0 / 12 SME routes | every screen; staging-only |
| SME_CONTENT_LANGUAGE_API.md | not at all (language axis) | 1 / 13 (English-only rule copy) | language control, `X-Lang` (proxy can't carry headers), all `…ByLanguage` editors |
| DEEP_LINKS.md | partial | 3 / 9 | `/open/paywall` preset → Home; share link lacks `exam`/`src`/`go.` host |
| PYQ_MOBILE_CONTRACT_2026-10.md | n/a (app contract) | — | unindexed; portal can't author its fields |
| rollout-and-verify.md §7–§9 | §7–§8 mostly portal-visible; §9 none | ~11 / 14 visible | no section for the 10-30 content-language rollout |
| README.md | mostly accurate | — | L68 contradicted; cover note, INGEST and rollout rows stale; PYQ contract unindexed; `pyq_open` undocumented |
| archive/WHAT_CHANGED_2026-07-23.md | archived correctly | — | points only at WHAT_CHANGED_2026-09-21 (resolves) |

**Settled claims from the brief:**
- **`examIds` required + `*`:** true on our side (`requireExamIds` :147–164). The portal enforces it on single-row create (no default, `*` = "All exams").
- **Excel import path:** keyless `/cms/{pyq,mains}/bulk` via `import_and_push`, **no `examIds`** → `upsc-cse`. The UI says so. The keyed bulk routes are proxied and unused.
- **No language UI anywhere:** no `X-Lang` sent, no `…ByLanguage` exposed, zero `/sme/i18n` consumers (control `/sme/users` 59 + 17).
- **Filter-config:** icons from `validIconKeys`, hidden, sortOrder, Optional flip and `?exam=` are all rendered.
- **Content push:** sends `examIds`; status values DRAFT/PUBLISHED/ARCHIVED; `by-source`/`reparse` unused.
- **Reels:** `/reels/bulk` sends no `examIds`/`language`; `DELETE /reels` is admin-gated in the UI and keyless upstream; the blog save shape matches.

## Contradictions

1. **CMS_ADMIN_API.md teaches `isVerified` writes on PYQ/Mains** (PATCH `{"difficulty":"HARD","isVerified":true}` D:393–396, D:483–486; workflow step 4 D:1202; create examples D:148, D:189; field tables D:829, D:860). Code: `UpdatePYQPaperDto`/`CreatePYQPaperDto` (and mains) have no `isVerified` or `isActive`, and the global pipe has `forbidNonWhitelisted: true` (`main.ts:249`) → `400 property isVerified should not exist`. INGEST §3.3 already records this for `/sme/content`; the CMS doc was never corrected.
2. **README:68 "question-quality ⚠️ still not exam-scoped"** vs `sme-content.controller.ts:28–30` (`?exam=` accepted, `meta.exam` echoed). **And the portal proxy drops `exam`** (M:5880–5905 forwards no `exam`), so the portal's question-quality screen is still unscoped even though we support it.
3. **SME_CONTENT_LANGUAGE_API.md:94 `GET /sme/blog`** vs `@Controller('sme/blogs')` (`sme-blog.controller.ts:30`).
4. **DEEP_LINKS §4 says unknown keys open Home; the portal offers `/open/paywall`** as a banner preset (`BannerEditor.tsx:445`). The app's `ROUTES` and legacy aliases have no `paywall` (`DeepLinkTarget.kt:463`, :476–486), so the banner opens Home.
5. **DEEP_LINKS §5 says publish on `go.` with `src` and `exam`.** The portal's only share-link builder uses `app.prepmonkey.com/open/offer/<code>` with neither (`Offers.tsx:85,263,1132`). DEEP_LINKS §5 also says question ids come "from the SME portal (`GET /sme/content/pyq?exam=…`)", a route the portal never calls.
6. **CONTENT_DOC §PATCH "If `rawMarkdown` is included: all existing questions are deleted and recreated"** vs `content-doc.service.ts:334` (only when the markdown changed). Neither doc nor UI says that recreation **cascade-deletes `user_question_attempts`** (`schema.prisma` `UserQuestionAttempt.question … onDelete: Cascade`). Portal re-sync always sends `rawMarkdown` + `status:"PUBLISHED"` (`content_push.py:106–115`), so an edited re-sync wipes learners' quiz answers, and re-publishes an archived doc without updating `remote_status` (M:2687–2697 vs M:2716).
7. **Portal copy vs staging code (true on prod until rollout):** `content.api.ts:43`, `Mains.tsx:30–31`, `Mains.tsx:597` say mains `marks` cannot be set; INGEST §3.5 + `MainsMarksDto` say it can (since 10-30). `Simulations.tsx:96` says learners may see an empty test; INGEST §3.6 + `simulation.service.ts:46` hide it. These must flip in the same release as the prod rollout.
8. **Several 10-30 "Changed" header lines** (INGEST, CMS, CONTENT_DOC, FILTER_CONFIG) describe behaviour that is **staging-only**. Prod probe 2026-11-03: `/i18n/languages` 404, PYQ `language:"ENGLISH"`. Only LANGUAGE:33–35 and README:3 say so; the per-doc headers do not.
9. **rollout-and-verify.md has no section for `20261030000000_content_lang`**, although WORKLOG:37 makes it a gated prod step. README:81 describes the doc as complete through §9.
10. **AUDIT §1.2's "`PATCH /sme/content/pyq/{id}` … before-snapshot via `GET /cms/{bank}/{id}`"** is confirmed (M:8078). AUDIT §2.2's "patch only as the before-snapshot" for `/cms/pyq` is slightly off: the before-snapshot is a **GET**; the `/cms` PATCH is never used.
11. **Proxy forwards params the backend ignores:** `GET /cms/psychometric/test-sets` forwards `isActive`, `search` (M:8781), but the controller reads only `page`, `limit` (`cms-psychometric.controller.ts:118–120`). Harmless today: no UI sends them.

## Processes stopped

None started (no servers, no background jobs, no SQL). Network calls: 4 read-only public HTTPS GETs (prod and staging `GET /api/v1/i18n/languages`, `GET /api/v1/cms/pyq?limit=1`). Temporary output (`/tmp/claude-502/i18n.json`, `scratchpad/c3probe/`) was deleted. Only file written: this report.
