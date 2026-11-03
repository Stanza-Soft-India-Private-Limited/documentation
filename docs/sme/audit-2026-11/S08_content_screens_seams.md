# S08 — Content screens + AI→backend save seams (`/simulations`, `/pyq`, `/mains`, `/psychometric`, `/content-edits`, `/reels`; content push, Excel import, `_logged_question_patch`, blog save)

Read-only verifier pass, 2026-11-03. `M:` = `upsc-document-generator/app/main.py` (portal HEAD `7b00daf`, 2026-10-27, `version="2.44.0"` M:121, matches the live footer "Portal · v2.44.0"); `U:` = `portal/admin-ui/src/`; `B:` = `backend/src/` (HEAD `5eee69c`). Live captures = `~/Desktop/projects/.playwright-mcp/live/<screen>_*` (prod, after the 14:13 IST backend deploy). Prod SQL: read-only via `scripts/staging-db.ts --prod` (allowed by this brief, overriding SHARED_RULES' "no SQL" line); internal-account exclusion applied where users are counted. Builds on C3 (not repeated: `PYQ_CONTENT_FIELDS` strip, Excel no `examIds`, content-doc cascade, psychometric keyless, `isVerified` doc error).

**Headline**
1. 🟠 **`/mains` "Marks" column is not marks.** It renders `item.tags[0]` (`U:pages/Mains.tsx` cell after the Question cell, `{item.tags && item.tags.length > 0 ? item.tags[0] : '-'}`). Prod: 6,403 of 7,466 rows have tags[0] = "Sociology Optional", "PSIR Optional"… so the column says "Sociology Optional" under "Marks"; and the real `marks` column is **10 on all 7,466 rows**, including 507 tagged "15M" and 3 tagged "12.5M". The Excel importer writes the marks into `tags` and never into `marks` (`excel_parser.py` Mains builder: `tags = [marks_raw]`). The editor hint "the marks field itself cannot be set through the API" (`Mains.tsx`, Tags `hint`; `content.api.ts:43`) is false since 14:13 today: `SmeUpdateMainsDto` = `UpdateMainsPaperDto ∩ MainsMarksDto` (`B:modules/sme/dto/sme-content-ingest.dto.ts:115–142`).
2. 🟠 **387 of 487 reel blogs show as "No blog / Add blog".** The page builds its blog map from ONE page of `GET /sme/blogs?page=1&limit=100` (`U:pages/Reels.tsx` ~L995–1003, `limit: 100`), which our side orders by `date desc` over ALL reels (`B:modules/reels/reels.service.ts:772–817`). Prod: 858 reels, 487 blogs, 100 in the newest 100, **387 beyond**. On those cards the modal does not load the existing blog (`enabled: hasBlog`, Reels.tsx:762) and "Create blog" POSTs → `reel_blogs.reel_id` unique → 400 "Unique constraint violation" (`B:common/filters/exception.filter.ts:174`), after a Dify generation was spent.
3. 🟠 **Five filter dropdowns cannot reach most of the data they filter.** PYQ subject list (`api.ts:1516–1521`) lacks CSAT (1,040 UPSC rows) and Mental Ability (275 APPSC rows); PYQ/Mains year list stops at 2014 (`api.ts:1514`) while 1,737 PYQ rows (back to 1984) and 561 Mains rows are older; Mains "Paper" offers GS 1–4 only (`MAINS_PAPERS`, api.ts:1522) while 6,403 rows (86 %) are optional papers; Psychometric "Type" offers 5 types (`PSYCHOMETRIC_TYPES`, api.ts:1523), all 2,000 rows are `Onboarding`; Psychometric difficulty offers EASY…EXPERT, 1,000 rows are `DIFFICULT`.
4. 🔴 (known, scope confirmed) **Every content screen except `/simulations` runs on keyless routes**, and the keyless surface includes `DELETE /reels` (every reel + every Mux asset + 487 blogs by cascade, `reels.service.ts` `deleteAllReels`, `prisma.reels.deleteMany({})`; `ReelBlog … onDelete: Cascade`, schema.prisma:470–480) and hard `DELETE /cms/pyq/:id` with no FK from `pyq_attempts` (schema.prisma:1173–1198): prod already holds **5,158 orphaned PYQ attempts on 2,962 vanished question ids** (Apr–Sep 2026; source not attributable — no delete is audited anywhere).
5. Brief contradictions: `/content-edits` live rendered **1 row** (not 0); `/reels` rendered **"Reels Gallery (858)"** (not 0). Details in §D.

---

## A. TRUST

### A.0 Data path per screen (verified)

| Screen | List call (live network file) | Proxy | Our route (auth) | Writes |
|---|---|---|---|---|
| `/simulations` | `GET /sme/content/simulations` (`simulations_network.txt:416`) | M:8320 `_sme_proxy` | `sme-content-ingest.controller.ts` `listSimulations` (key) | POST/PATCH/bulk/assign keyed M:8329–8366; lookup M:8409→`GET /cms/simulations/questions` and soft delete M:8451→`DELETE /cms/simulations/:id` via `_cms_sim_request` M:8392 (**none**); GET-by-id derived from list M:8436–8448 |
| `/pyq` | `GET /cms/pyq?page=1&limit=20` (`pyq_network.txt:429`) | M:8464 → `cms_proxy.list_pyqs` :94 | `cms-pyq.controller.ts:55` (**none**) | create `POST /sme/content/pyq` (key); edit `PATCH /sme/content/pyq/:id` via `_logged_question_patch` M:8066 (before-read keyless `GET /cms/pyq/:id` M:8078); delete `DELETE /cms/pyq/:id` M:8558 (**none**); Excel M:8506 → `/cms/pyq/bulk` (**none**) |
| `/mains` | `GET /cms/mains?page=1&limit=20` (`mains_network.txt:442`) | M:~8575 → cms_proxy :132 | `cms-mains.controller.ts:55` (**none**) | same pattern (M:8572–8668) |
| `/psychometric` | `GET /cms/psychometric/questions?page=1&limit=20` (`psychometric_network.txt:454`) | M:8683ff | `cms-psychometric.controller.ts` 11 routes (**none**) | all keyless |
| `/content-edits` | `GET /content-edits?days=30&page=1&limit=50` (`content_edits_network.txt:464`) | M:8103 — **their DynamoDB** (`app/services/content_edits.py:1–20`) | — (only `answer-mismatches` M:8167 reaches `/cms/pyq` 48× pages) | none |
| `/reels` | `GET /reels?page=1&limit=20`, `GET /sme/blogs?page=1&limit=100`, `/reels/views` (Mux, out of scope) (`reels_network.txt:476–478`) | M:4096 (plain httpx, no key), M:6244 (key) | `reels.controller.ts:61` `@Public() @Feature('reels')` (**none**); `sme-blog.controller.ts` (key) | `PUT /reels/bulk` M:4152, `DELETE /reels/:id` M:4196, `DELETE /reels` M:4226 (admin in portal), `POST /reels/:id/like` M:4259 (**no such route on our side → 404**; only `reels-user.controller.ts` has feed/bookmark/:id), blogs POST/PATCH/DELETE keyed M:6265–6301 |
| Content push (`/content/:id`) | — | M:2659 → `content_push.py:78` POST / :123 PATCH; M:~2722 → :145 PATCH status | `content-doc-admin.controller.ts:35, :94` (**none**) | see A.5 |

Bug-class sweep for these callers: (1) params-replace — none (`cms_proxy._make_request` builds URL from a bare path, cms_proxy.py:54–64; `_sme_proxy` merges, M:4356–4365); (2) `_sme_fetch_all` — not used here; (3) fetch-then-lookup — **the blog map (Headline 2)**; (7) `dict(request.query_params)` on `/sme/content/{pyq,mains,simulations}` GET (M:8250, 8288, 8326) — single-valued params only, harmless; (8) answer-mismatch `complete` DOES compare `len(rows) >= total` (M:8213) — correct; (9) mislabels — Mains "Marks", PYQ "Duration (seconds)", reel "likes"; (10) none found.

### A.1 Per-number table

| # | Number as labelled (UI file:line / live) | Plane | Our definition | Their transformation | Window/TZ | Exclusions | Verdict | Prod SQL (run, §E) | Sev |
|---|---|---|---|---|---|---|---|---|---|
| 1 | `/simulations` pills "All 6 · Active 6 · Inactive 0" (`Simulations.tsx:242–244`; live `simulations_snapshot.yml`) | A (content) | `simulation.findAll(exam, lang)` keyed list, inactive included | client counts `rows.filter(isActive)` :153–157 | all time | n/a | TRUE (6 rows, all active, all `upsc-cse`, all `en`) | E2 | ⚪ |
| 2 | "6 active tests with 587 assigned questions in total (… counts twice)" (:259–263) | A | `simulations.totalQuestions` (set by assign) | sum over active rows | — | — | TRUE: 87 + 5×100 = 587; `totalQuestions` = link count on all 6 (E9) | E9 | ⚪ |
| 3 | Row "87 questions · 105 min · +2 / -0.67 / 0 · #1" (:84–93) | A | columns as stored (`wrongMark` −0.6666667) | `fmtMark` rounds to 2 dp | — | — | TRUE | E9 | ⚪ |
| 4 | "Active with no questions. Learners may see an empty test." (:94–98) | — | since 10-30 code the app hides active sims with 0 questions (`simulation.service.ts:35–46`, C3 INGEST #24) | — | — | — | STALE copy (now prod per §5A.1); never fires today (0 empty sims) | — | 🟡 |
| 5 | `/pyq` "· 4707 total questions" + "Questions (4707)" (`PYQ.tsx` header; live `pyq_snapshot.yml:127`) | A | `count(pyq_papers where …)` via `PaginationUtil` (`cms-pyq.service.ts:112–167`) — all languages, all active states | `meta.total` | — | none (no `isActive` filter sent; all rows active anyway) | TRUE: 2,641 UPSC + 2,066 APPSC (E2) | E2 | ⚪ |
| 6 | PYQ "Verified ✓ / Unverified" column (`PYQ.tsx` Verified cell) | A | `isVerified` — not writable by anyone via the portal (DTO lacks it, `content.api.ts:9–12`) | — | — | — | TRUE but meaningless for UPSC: 2,009 of 2,066 APPSC verified (the 10-01 script), 0 of 2,641 UPSC. The column tells a viewer which bank was loaded by script, not which question a human checked | E4 | 🟡 |
| 7 | PYQ editor "Correct Answer" select A–D, shows **A** when the row has none (`PYQ.tsx` `value={formData.correctAnswer || 'A'}`) | A | `correctAnswer` A–E, null only when `boardDeleted` (`pyq-withdrawn.util.ts:89–118`) | fallback to 'A' | — | — | MISLABELLED for 17 APPSC board-withdrawn rows (E4): they display answer "A" | E4 | 🟠 |
| 8 | PYQ editor "Duration (seconds)", default 72 (`PYQ.tsx:706`, DEFAULT_FORM `duration: 72` :76) | A | `pyq_papers.duration` — schema comment "Duration in minutes" (schema.prisma:282); prod values only 120 (3,214) and 150 (1,493) = the paper length in minutes (UPSC 120, APPSC 150) | — | — | — | MISLABELLED: a portal-created question gets 72 in a column whose every other row is paper-minutes; Excel defaults 120 (`excel_parser.py` PYQ `duration … or 120`). Doc examples also use 72 (INGEST:301, 339) | E5 | 🟡 |
| 9 | `/mains` "Questions (7466)" (live `mains_snapshot.yml` heading) | A | count `mains_papers` | `meta.total` | — | — | TRUE (all `upsc-cse`, `en`) | E2 | ⚪ |
| 10 | **Mains "Marks" column** ("15M", "10M", "Sociology Optional"…) (`Mains.tsx` Marks cell = `item.tags[0]`; live `mains_snapshot.yml:70,85,100,114`) | A | `mains_papers.marks` Int default 10 (schema.prisma:349) | renders the first TAG | — | — | **WRONG**: 6,403 rows show an optional-paper name; `marks` = 10 on every row incl. the 507 tagged "15M" | E3 | 🟠 |
| 11 | Mains "Paper" column (= `subject`) | A | `subject` | — | — | — | TRUE ("GS Paper 1" or "<X> Optional") | E10 | ⚪ |
| 12 | `/psychometric` "Psychometric Questions (2000)" (live `psychometric_snapshot.yml`) | A | count `psychometric_questions` | — | — | — | TRUE | E7 | ⚪ |
| 13 | Psychometric "Type" / "Difficulty" cells and editor selects (`Psychometric.tsx` ~L356–440) | A | free strings: all 2,000 `psychometricType='Onboarding'`, difficulty `EASY`×1,000 / `DIFFICULT`×1,000 | editor selects only list `PSYCHOMETRIC_TYPES` and `CMS_DIFFICULTIES` | — | — | MISLABELLED in the editor: a controlled `<select>` whose value is not an option shows "Select type" / a wrong difficulty, while the whole-row PATCH re-sends the stored value; touching the select silently changes data. Filters cannot select either value (Headline 3) | E7 | 🟡 |
| 14 | Psychometric "Correct Answer" select defaults 'A' (`correctAnswer: item.correctAnswer || 'A'`) | A | nullable | whole-row PATCH (`usePsychometric.ts:51`) | — | — | latent: today 0 rows have a null answer (E7), so nothing is overwritten | E7 | ⚪ |
| 15 | Test Sets "N questions" (`Psychometric.tsx` test-set table `ts.questionIds?.length`) | A | `questionIds` array length | — | — | — | TRUE: 100 sets × 20 | E7 | ⚪ |
| 16 | `/content-edits` StatChips "Edits saved 1 · Questions touched 1 · Failed saves 0 · <email> 1" (`ContentEdits.tsx:105–109`; live `content_edits_snapshot.yml:157–169`) | their DynamoDB | — (our `sme_audit_log` CONTENT_INGEST has route/id only, no actor/reason/diff, `sme-content-ingest.controller.ts` `recordIngest`) | `content_edits.summarise` | last 30 d, `editedAt` UTC ISO | none (the one editor is staff — appropriate here) | TRUE for their log; our audit shows 6 PATCHes since 10-22 (10-22 12:19, 10-27 13:00/13:32/13:59/13:59, 10-28 11:54 IST). The 10-28 one is the logged row; the 10-27 ones fall around the v2.44 merge (13:50 IST) — whether they predate the log or failed to log is UNKNOWABLE from our side | E8 | ⚪ |
| 17 | Answer-mismatch tab "Mismatches · Questions scanned · Incomplete" (`ContentEdits.tsx:209–216`) | A | full `/cms/pyq` scan, 100/page, sem 6 (M:8167–8215) | `complete = no failed pages and rows ≥ total` | live | — | TRUE (completeness is checked properly). Not run live (the capture did not open the tab) | — | ⚪ |
| 18 | `/reels` "Reels Gallery (858)" (`Reels.tsx:599`; live `reels_snapshot.yml:134`) | A | count `reel_items` (no exam scope: portal sends no exam, `ReelsFiltersDto` has none) | `meta.total` (portal re-models rows through `Reel(**r)`, models.py:477–493, which drops `examIds`/`language`) | — | — | TRUE: 858, all READY, all `examIds=['*']`, all `en` | E2, E11 | ⚪ |
| 19 | Reel card "Has blog / No blog", "Edit blog / Add blog" (`Reels.tsx:473–500`, `hasBlog={!!blogMap[reel.id]?.hasBlog}` :690) | A | `reel_blogs` 1:1 | map from first 100 reels by date | — | — | **WRONG for 387 reels** | E11 | 🟠 |
| 20 | Reel like button count (`Reels.tsx:486–490` `{reel.likes}`) | — | no likes column, no like route on our side | `likes: int = 0` default (models.py:490); POST → 404 | — | — | WRONG: always 0, the button cannot work | — | 🟡 |
| 21 | Reel views / watch time (Mux Data) | — | out of scope (Mux) | — | 30 d | — | not judged | — | — |

### A.2 Every form field of every editor

Legend: ✅ path intact · ✂ stripped/never sent · ⛔ we reject · 👻 we accept, the form cannot set.

**PYQ editor** (`PYQ.tsx` drawer; `savePyq` → `content.api.ts:141–181`; create `POST /sme/content/pyq` M:8253 → `SmeCreatePyqDto`; edit `PATCH /sme/content/pyq/:id` M:8271 → `_logged_question_patch` → `UpdatePYQPaperDto` (generated) → `pyq_papers`)

| UI field | Payload key | Our DTO / column | Notes |
|---|---|---|---|
| Edit reason + note (edit only, `EditReasonFields`) | `_edit` | popped by the portal M:8068, never reaches us | their log only |
| Exams (`ExamIdsPicker`, required on create) | `examIds` | `RequiredExamIdsDto` (create) / optional on PATCH, strict (`requireExamIds` controller) → `examIds` | ✅ "All exams" = `*` |
| Exam Name | `examName` | ✅ | free text |
| Year (select 2026…2014) | `year` | ✅ | rows < 2014 open as "Select Year" (not sent unless touched) |
| Subject (datalist, free text) | `subject` | ✅ | |
| Topic | `topic` | ✅ | |
| Question (markdown) | `question` | ✅ | |
| Option A–D | `optionA..D` | ✅ | |
| — | `optionE` | 👻 in `PYQ_CONTENT_FIELDS` but no input; `openEdit` never loads it | 0 prod rows use E (E4) |
| Correct Answer A–D | `correctAnswer` | ✅ (`assertPyqAnswerKey`) | E not offered; null shown as A (A.1 #7) |
| Difficulty / Source / Nature / Total marks | same | ✅ enums | |
| Duration (seconds) | `duration` | ✅ | unit mislabelled (A.1 #8) |
| Tags / Keywords / Related topics | same | ✅ required non-empty on create | |
| Answer Explanation, Solve Tip | same | ✅ | `AnswerMismatchWarning` client-side |
| QuestionFlags (read-only `isActive`, `isVerified`) | — | ✂ by design; our DTO has neither (⛔ if sent, `main.ts:249` forbidNonWhitelisted) | correct |
| — | `paperNumber`, `questionNumber`, `weightage`, `markingScheme`, `prediction` | 👻 | |
| — | `paperLabel`, `questionImageUrl`, `imageCaption`, `boardDeleted` | 👻 and ✂ (`pick()` drops them, content.api.ts:36–41) | prod: `paperLabel` on all 2,066 APPSC rows, figures on 38, 17 withdrawn — none visible or editable |
| — | `language` | 👻 strict on keyed (`requireLanguage`/`optionalLanguage`) | prod catalogue has only `en` (E1) |
| Delete (trash icon, `aria-label="Delete"`, two clicks within 3 s + toast) | — | `DELETE /cms/pyq/:id` **hard** (`cms-pyq.service.ts:200–207`), no audit, no attempts cleanup | content reviewer hidden (`!reviewer`) |

**Mains editor** (`Mains.tsx`; `saveMains`; `SmeCreateMainsDto` / `SmeUpdateMainsDto`)

| UI field | Key | Ours | Notes |
|---|---|---|---|
| Reason, Exams, Exam Name, Year, Paper/subject, Topic, Question, Model Answer (`answer`), Approach (`answerExplanation`), Tips (`solveTip`), Difficulty, Tags, Keywords, Related topics | same | ✅ | "Approach"/"Tips" labels rename `answerExplanation`/`solveTip` — fine |
| — | `marks` | 👻 **accepted since 14:13** (`MainsMarksDto` 1–50); ✂ by `MAINS_CONTENT_FIELDS` (content.api.ts:44–48) | copy says it cannot be set (live `mains_modal_edit_snapshot.yml:747`) |
| — | `isOptional` | 👻 in `MAINS_CONTENT_FIELDS`, no control (derived from " Optional" suffix on create only) | the Optional↔GS flip lives on `/filter-config` |
| — | `questionNumber`, `prediction`, `language` | 👻 | |
| Row Edit / Delete icons | — | **no `aria-label`** (live `mains_snapshot.yml:75–76` "button [ref]") ; Delete = two clicks within 3 s, hard `DELETE /cms/mains/:id` | `mains_evaluations` has no FK either (0 orphans today, E6) |

**Simulation editor** (`SimulationFormDrawer.tsx`; create body :118–129, PATCH diff :143–155; `SmeCreateSimulationDto` / `UpdateSimulationDto` = `PartialType(CreateSimulationDto)`)

| UI field | Key | Ours |
|---|---|---|
| Exams chips + "Every exam (*)" | `examIds` | ✅ strict |
| Name, Description, Duration (minutes), Sort order, Correct/Wrong/Skipped marks, Card colour, "Active (visible in app)" | `name, description, durationMinutes, sortOrder, correctMark, wrongMark, skippedMark, color, isActive` | ✅ all |
| — | `totalQuestions` | 👻 accepted by `CreateSimulationDto` (`@IsInt @Min(0)`) — correctly never sent (set by assign) |
| — | `language` | 👻 |
| Deactivate (ConfirmModal, `Simulations.tsx:323–338`) / Reactivate (no confirm, harmless) | `DELETE /cms/simulations/:id` soft / PATCH `isActive:true` | ✅ |
| Questions drawer: bulk import JSON/CSV (`questionImport.ts` validates A–D uppercase), lookup, "Replace question set (n → m)…" with ConfirmModal | `/sme/content/simulations/questions/bulk`, `/cms/simulations/questions`, `/assign-questions` | ✅; `isVerified`/`isActive` per question are accepted and parsed (questionImport.ts:12) |

**Psychometric question editor** (`Psychometric.tsx` QuestionsTab; whole-form PATCH, not a diff)

| UI field | Key | Ours (`modules/psychometric/dto/create-question.dto.ts`) |
|---|---|---|
| Subject (select of UPSC subjects), Topic, Psychometric Type (select), Question, Option A–D, Correct Answer (A–D), Difficulty (EASY…EXPERT), Active, Verified, Tags, Keywords | same | ✅ all writable (incl. `isActive`, `isVerified`) |
| — | `questionNumber` | 👻 |
| — | `examIds`, `language` | **absent from the table** (schema.prisma:398–417) |
| Delete (`title="Delete"`, two clicks 3 s) | hard `DELETE /cms/psychometric/questions/:id` | test sets keep the dangling id in `questionIds[]` (no FK) |

**Test set editor**: Name, Description, Question IDs (comma text), Active → `CreateTestSetDto` ✅; 👻 `language` (column exists, `psychometric_test_sets.language`, all `en`). List proxy forwards `isActive`/`search` we ignore (M:~8781 vs controller :118–120, C3).

**Reels** (`Reels.tsx`): upload queue fields Title, Description, Subject (UPSC list), Date → `PUT /reels/bulk` {id,title,description,subject,context,date,assetId,playbackId} (`Reels.tsx:1135–1144`) → `bulk-update-reels.dto.ts`; 👻 `examIds` (defaults `['*']`, `reels.service.ts:465`), `language`, `status`. Delete reel = trash, two clicks 3 s; **Delete All** = admin-only button, two clicks 5 s (`Reels.tsx` `handleDeleteAll`), keyless upstream. Like = 404.

**Blog editor** (Reels "Add/Edit blog" modal): Title, Markdown (from Dify generate — out of scope) → `POST /sme/blogs {reelId,title,rawMarkdown}` / `PATCH {title,rawMarkdown}` (allow-listed M:6265–6292) → `CreateSmeBlogDto`/`UpdateSmeBlogDto` (`sme-blog.dto.ts:41–70`) → `reel_blogs` + parsed `sections`. ✅ shape. "Remove this blog?" uses `window.confirm` (Reels.tsx:960). Audited `BLOG_CREATE/UPDATE/DELETE` (no actor). Prod: 496 creates, 454 of 470 followed by an update within 10 s (E8) — a double save per blog, harmless.

**Content push** (`ContentEditor.tsx:298–340`): Push/Re-sync button with `window.confirm` (:302–304); status select DRAFT/PUBLISHED/ARCHIVED fires `changeRemoteStatus` on change **with no confirmation** (:324–340, :567–573) and shows DRAFT when `remote_status` is unknown.

### A.3 Seams

| Seam | Code | What reaches us | Verdict |
|---|---|---|---|
| **`_logged_question_patch`** | M:8066–8100 | `_edit` popped; before = keyless `GET /cms/{bank}/{id}`; PATCH keyed; their DynamoDB row per attempt incl. failures | ✅ works. Our side records `CONTENT_INGEST {route, count, examIds, id, language?}` with **no actor, reason or diff** — the person-level log exists only in their DynamoDB. The before-read is the only reason this seam still needs a keyless GET. |
| **Excel import** | M:8506–8517 (PYQ), M:~8609–8620 (Mains) → `excel_parser.py` → `import_and_push` (`excel_parser.py:~306`, batches of 100) → keyless `/cms/{bank}/bulk` | PYQ items: no `examIds`/`language` (→ `upsc-cse`, `en`); `correctAnswer` raw (`_safe_str`, a lowercase/blank cell 400s its whole batch via `assertPyqAnswerKey`, now prefixed `items[i]`); **missing year → 0** (`_safe_int(None) = 0`, accepted); `duration … or 120`. Mains: only sheets GS1–GS3 (`_MAINS_SHEET_SUBJECTS`), `examName` hard-coded "UPSC CSE Mains", marks → `tags`, `marks` never set. Earlier batches stay written when a later one fails; re-import duplicates (no dedupe key). Result toast "Import complete: X succeeded, Y failed" is a success toast even when all failed (`usePYQ.ts:88`); per-batch errors not rendered | 🟠 |
| **Content push** | `content_push.py:46–150`; M:2659–2740 | POST {title, subject, topic, subTopic, rawMarkdown, status PUBLISHED, language "ENGLISH", sourceId, examIds}; PATCH re-sync always the same minus language, **always `status:PUBLISHED`** (:112) | cascade + un-archive confirmed (C3; `content-doc.service.ts:334–339`, delete at :335). New this pass: (a) every re-sync **regenerates the slug** (`generateSlug` appends `Date.now()`, :1244–1253, called whenever `title` is sent, :386/:415) — cosmetic, the app reads by id; (b) the 404 fallback re-POSTs on any error text containing "404" (M:~2697 `if "404" in err_str`). Prod: 530 PUBLISHED / 5 DRAFT / 0 ARCHIVED, 0 duplicate `source_id`, **200 documents had quiz questions recreated after creation** (10,358 rows) — how many learner answers were deleted is UNKNOWABLE (deleted rows leave no trace; progress counters stay consistent, E12) | 🔴 (known) |
| **Blog save after Dify** | `Reels.tsx:773–788` → M:6265/6284 → `sme-blog.controller.ts:50–75` | `{reelId,title,rawMarkdown}` | ✅ shape; ✗ wrong create-vs-update choice for 387 reels (Headline 2) |

### A.4 Keyless → keyed parity matrix (owner ruling: replicate end to end on `/sme/*` before the portal switches)

| Keyless route the portal calls | Portal callers | Keyed `/sme/*` today | Lacks | End-to-end parity must cover |
|---|---|---|---|---|
| `GET /cms/pyq`, `GET /cms/mains` (page, limit, exam, subject, year, difficulty, search, isActive, isVerified) | lists (`api.ts:1527, 1568`), answer-mismatch scan M:8178, filter-config shared-sample M:6529 | `GET /sme/content/pyq|mains` — SAME service + filters (`CmsPyqFiltersDto`), + `X-Lang` (controller `listPyq/listMains`); proxied M:8244/8282, unused | nothing on our side | identical `{data, meta{total,page,limit,totalPages,hasNext,hasPrev}}` (same `PaginationUtil`); a way to pass language (the proxy cannot carry `X-Lang`, M:4369 — accept `?language=` as a query alias, or the proxy must forward a header); filter-config sample + mismatch scan switch too |
| `GET /cms/pyq/:id`, `GET /cms/mains/:id` | `?edit=` deep link (`usePYQ`/`useMains`), feedback report target M:6717, **`_logged_question_patch` before-read M:8078** | none | GET-by-id | same row shape incl. `paperLabel/questionImageUrl/imageCaption/boardDeleted/language/marks`; 404 `{success:false,message}`; ParseUUID 400 |
| `DELETE /cms/pyq/:id`, `DELETE /cms/mains/:id` (hard) | trash icons | none | DELETE | audit row (who/what/before snapshot); decide hard vs soft (`isActive=false`) — hard today orphans `pyq_attempts`/`mains_evaluations` (no FK) and Neo4j relationships; the portal should also log deletes in its edit log (today it does not) |
| `POST /cms/pyq/bulk`, `/cms/mains/bulk` (Excel) | `import_and_push` | `POST /sme/content/{pyq,mains}/bulk` exists (1–100, all-or-nothing, examIds + strict language per item) | nothing | the Excel parser must send `examIds` (an exam picker on the import dialog), `language`, uppercase letters, reject year 0, mains `marks` from the Marks column; error shape: keyed batch 400 names `items[i]` |
| `POST/PATCH /cms/pyq|mains` | dead UI fns (`createPYQ`, `updatePYQ`, …) | keyed exist and are used | — | retire the proxies M:8521, 8545 and mains twins |
| `/cms/psychometric/questions[/bulk|/:id]` ×6, `/cms/psychometric/test-sets[/:id]` ×5 | whole `/psychometric` page | **none** | all 11 | same filters (subject, topic, psychometricType, difficulty, search, isActive, isVerified) and pagination; test-set list should actually honour `isActive`/`search` or the proxy stop sending them; audit; `language` on test sets (column exists) and a decision whether questions get `language` (no column today) |
| `GET /cms/simulations/questions?externalIds` | sim drawer lookup M:8409 | none | lookup | ≤500 ids, matches only, order not guaranteed |
| `DELETE /cms/simulations/:id` (soft) | Deactivate M:8451 | none | DELETE | soft semantics, audit |
| (derived) `GET /sme/content/simulations/:id` | `getSimulation` unused; feedback doc cites it (`SME_FEEDBACK_API.md:186`) | none (portal fakes it from the list M:8436) | GET-by-id | — |
| `POST/PATCH/GET /content-doc-admin[/:id]` | content push, status change, feedback target M:6614–6617 | **none** | all | create/patch/get + `by-source` idempotency; strict `examIds`/`language` like `/sme/content`; **the re-sync must not cascade-delete attempts or force PUBLISHED** (parity must not copy that behaviour); audit |
| `GET /reels`, `PUT /reels/bulk`, `DELETE /reels/:id`, `DELETE /reels`, `POST /videos/upload-url`, `GET /videos/:id/status` | `/reels` | **none** (only `/sme/blogs`) | all | list must not sit behind `@Feature('reels')`; `examIds`/`language` in list rows and bulk body; `DELETE /reels` (all) should NOT be mirrored (or only behind an explicit confirm token); the two `/videos/*` routes are the Mux seam — row creation is ours, upload is out of scope |
| `POST /reels/:id/like` | like button | — | — | **does not exist on our side**; drop the button, do not build parity |
| `/reel-blog-admin/*` | none | `/sme/blogs` supersedes | — | retire, no parity |
| `/content-edits` | — | their DynamoDB, not ours | — | out of scope (their store). Our `CONTENT_INGEST` audit has no actor; if the owner wants the edit log on our side, the keyed PATCH would need to accept the portal's `_edit` (reason/note/reportId + an asserted actor) — flag only, needs a decision because the key is shared |

### A.5 Exam + language scoping per screen

| Screen | Exam control | Exam in rows | Language |
|---|---|---|---|
| `/simulations` | `ExamSelector` → keyed `?exam=` (matches exam OR `*`), copy says so (:254–258) | chips per test, "Every exam" | none; list mixes languages (all `en` today) |
| `/pyq`, `/mains` | `ExamSelector` in URL → `/cms/*?exam=` (validated, 400 on unknown) + info banner | "Exams" chips column; editor `ExamIdsPicker` | none; `/cms` list has no language filter at all, `X-Lang` cannot be proxied |
| Excel import (both) | **none** — everything lands in `upsc-cse` (warned in UI `PYQ.tsx:150`, `Mains.tsx:277`) | — | none → `en` |
| `/psychometric` | none; the table has no exam column (opt-out is the exam's `psychometric` feature flag, INGEST §7) | — | questions have no column; test sets have `language`, not shown |
| `/content-edits` | log: none; mismatch scan: `ExamSelector` → `exam` forwarded | — | none |
| `/reels` | none; rows' `examIds` dropped by the portal model | — | none (reels are language-blind by ruling B23, `reels.service.ts:340–343`) |
| Content push | `ContentExamPanel` → `exam_ids`, read-back of stored tags | ✅ | hard-coded `"ENGLISH"` |

"All exams" vs what exists (E2): PYQ 4,707 = 2,641 UPSC + 2,066 APPSC (no `*` rows) ✓; Mains 7,466 all UPSC (APPSC selection = 0, a true answer) ✓; Simulations 6 all UPSC ✓; Psychometric 2,000 (no exam axis) ✓; Reels 858 all `*` ✓; content docs 353 UPSC + 182 APPSC. `ExamSelector` lists exactly the two active exams (live) ✓. Language after today's migration: **every row in every bank is `en`, and the prod catalogue `i18n_languages` holds only `en`** (E1) — a Hindi write via `/sme/content` would 400 until `hi` is added.

---

## B. MEANING

> Owner ruling (quoted): "The numbers are there BUT those numbers are not explaining anything — same on the aspirant profile screen. No need to drastically change stuff; compact the details or combine and show. I don't need to see the endpoints a user is hitting on the portal, it explains nothing. Instead explain in simple terms: some X metric done on this data, or 'he stayed X mins', or 'he has been using this many times'. I may be grossly oversimplifying — the portal might already do some things well and corrections may not be needed at all. Just analyze and judge."
>
> Reference ruling: `/insights/purchase-funnel` is the house template — one-line subtitle, a plain scope line, plain counts, one ratio with its caveat, our-fault vs theirs split, the standard exam selector (+ language) as the scoping controls.

These are authoring screens, not analytics: shareholder questions (1) feature use and (4) question quality are the only ones they touch; (2) per-exam/per-language funnels only through scoping; (3), (5), (6) not at all.

| Panel | Plain sentence today | Judgement | Proposal (minimal) | Computable today? |
|---|---|---|---|---|
| `/simulations` header + pills + "6 active tests with 587 assigned questions…" | "6 UPSC mock tests are live, 587 questions in total, none for APPSC." | **KEEP** — the best content screen: build-order strip, honest scope line, confirm on deactivate, replace-warning on assign | add one usage clause per test row: "392 people started, 113 finished, last taken today" | yes: `simulation_attempts` (`userId`, `startedAt`, `submittedAt`), internal accounts excluded via `user_auth.email` (E9b) |
| `/pyq` header "4707 total questions" + Questions card | "4,707 prelims questions: 2,641 UPSC, 2,066 APPSC." | **COMPACT** into one scope line in the purchase-funnel style | "Showing 2,066 APPSC prelims questions in English · 17 withdrawn by the board · 2,009 script-verified" — exam + language selectors as the standard controls | yes: `pyq_papers.examIds/language/boardDeleted/isVerified` |
| PYQ "Verified" column | none ("Unverified" on every UPSC row says nothing) | **RAW/NOISE** for UPSC | fold into the scope line count; keep the filter | yes |
| PYQ filters (year/subject/difficulty/status) | — | **fix, not redesign**: options must come from the data (distinct subjects/years for the selected exam) | `GET /sme/filter-config/prelims` already returns per-subject entries for an exam (`/filter-config` uses it) | yes |
| `/mains` "Questions (7466)" + "Marks" column | "7,466 UPSC mains questions; 6,403 are optional-paper questions." — the Marks column cannot be read | **COMPACT** scope line ("Showing 7,466 UPSC mains questions in English · 1,063 GS · 6,403 optional") + **replace** Marks cell with `marks` and expose `marks` in the editor | after a one-off backfill of `marks` from the "NM" tags (507 + 309 + 3 rows) | `isOptional`, `marks` yes; the backfill is a data fix |
| `/psychometric` "(2000)" + Test Sets | "2,000 onboarding questions in 100 sets of 20, English only." | **COMPACT** to that sentence; the Type/Difficulty filters are noise until their options match the data | — | yes |
| `/content-edits` StatChips | "1 question edited in the last 30 days, by 1 person, 0 failed." | **KEEP** (their store; already a sentence-like set of chips) | — | n/a (their DynamoDB) |
| Answer-mismatch tab | "N questions' explanations contradict their answer key." | **KEEP** — directly answers question (4) | add exam to the scope line (already has the selector) | yes |
| `/reels` "Reels Gallery (858)" + likes + views | "858 reels, all shared by every exam; 487 have a blog." | **COMPACT**: scope line "858 reels for every exam · 487 with a blog"; **RAW**: drop the like count/button (no backend) | — | yes (`reel_items`, `reel_blogs`) |
| Content editor push block (status, synced, needs re-sync) | "Pushed to the app as PUBLISHED for UPSC, last synced 2 Nov." | **KEEP**, with the re-sync confirm saying what it destroys ("learners' quiz answers on this document are reset if the text changed") | — | the warning needs no data |

What each editor needs for multi-exam + multi-language (minimal):
- **PYQ / Mains / Simulation editors:** exams already first-class. Add ONE language dropdown fed by the catalogue (SME_CONTENT_LANGUAGE_API brief) and add `language` (+ mains `marks`, the four PYQ fields) to the `*_CONTENT_FIELDS` allow-lists; lists need a language selector next to the exam selector, which needs the proxy to pass a language (header or query alias). Today the dropdown would hold one value (`en`) — build it, but it is cosmetic until `hi` is catalogued on prod.
- **Excel import:** an exam picker + language picker on the import dialog, sent per item, via the keyed bulk route.
- **Psychometric:** questions have neither column; test sets have `language`. Minimal = show/set the test-set language; whether questions need a language column is an owner decision (flag only).
- **Reels:** exam is `*` everywhere and language is blind by ruling B23 → no change needed beyond not dropping the fields in the portal model.
- **Content push:** replace `"language": "ENGLISH"` with the item's language.

No new persistence is needed for any sentence above except a person-attributed edit log on our side (flag only) and per-question delete history (flag only).

---

## C. GAP (docs → proxy → UI)

| Item (doc) | Status |
|---|---|
| `GET /sme/content/{pyq,mains}` lists (INGEST §3.4) | proxied-only (M:8244, 8282); UI lists use `/cms` |
| `POST /sme/content/{pyq,mains}/bulk` (INGEST §3.2) | proxied-only; Excel uses keyless `/cms/*/bulk` |
| mains `marks` (INGEST §3.5, 10-30, live since 14:13) | absent in UI (stripped `content.api.ts:44–48`); UI copy says impossible — **stale** (`Mains.tsx` hint, `content.api.ts:43`, Mains `EMPTY_FORM` comment) |
| `language` + `X-Lang` (LANGUAGE §1–2) | absent; proxy structurally cannot forward the header |
| PYQ `paperLabel`, `questionImageUrl`, `imageCaption`, `boardDeleted`, `pyq-figures` upload (INGEST §9) | absent (C3 #28–36); prod data exists for APPSC (E4) |
| `optionE` (INGEST field list) | in allow-list, no input — absent |
| empty-simulation hiding (INGEST §3.6) | UI warning copy stale (`Simulations.tsx:96`) |
| `/content-doc-admin` list / `by-source` / `reparse` / DELETE (CONTENT_DOC) | absent (script tools) |
| `PUT /reels/bulk` `examIds`/`language` (INGEST §7, LANGUAGE §1.2) | absent in body (`Reels.tsx:1135–1144`); fields also dropped on read by `Reel` model (models.py:477–493) |
| `POST /reels/:id/like` (portal M:4259) | **route absent on our side** → 404 |
| `/reel-blog-admin/*` (CMS docs) | absent in portal (superseded by `/sme/blogs`) — retire |
| `GET /sme/content/simulations/:id` (SME_FEEDBACK_API.md:186) | does not exist on our side; portal derives it |
| `GET /sme/blogs` `hasBlog`/`search` (sme-blog.dto) | proxied, UI never sends `hasBlog=true` — the fix for Headline 2 is one param |
| Test-set list `isActive`/`search` (proxy M:~8781) | proxied, ignored by us |

### C.1 Windowed routes: what a standard Exam + window pills + custom from–to + Refresh control would need (owner ruling 2026-11-03)

| Screen / panel | Route chain | Window params our backend accepts (DTO, cap) | What the proxy forwards | What the UI sends / offers | Source + retention | Exam scoping | Gap for the standard control |
|---|---|---|---|---|---|---|---|
| `/content-edits` Edits tab | `GET /content-edits` M:8103 → **their DynamoDB** (`content_edits.load_rows`, `content_edits.py:149–155`) — never reaches us | n/a (not our backend). Portal route: `days` int 1–365 (M:8105 `Query(ge=1, le=365)`); no from/to | — | `DAY_OPTIONS = [7, 30, 90, 365]` select (`ContentEdits.tsx:36, 71–72`); Refresh = none (react-query refetch only on filter change); no custom range | DynamoDB `upsc-agent-settings`, rows from v2.44.0 (deploy 2026-10-27) onward; no TTL in code (0 hits for ttl/expire in `content_edits.py`), so effectively permanent | **none** (rows carry bank + contentId, not exam) | from/to would be a portal-side change (rolling `now − N days` in UTC, `since_days` :193 — not IST calendar days); exam filter needs the question's `examIds` stored in the row or a join. Our-side equivalent: `sme_audit_log` CONTENT_INGEST (`created_at`, permanent, no actor) — no read API exists for it |
| `/content-edits` Answer-mismatch tab | `GET /content-edits/answer-mismatches?exam=` M:8167 → `/cms/pyq` pages (keyless) | no window (a snapshot of the current bank) | `exam` only | `ExamSelector` + "Scan" button (= Refresh) | `pyq_papers`, current state | ✅ exam | window not meaningful here; keep exam + Refresh only |
| `/reels` gallery | `GET /reels` M:4096 → `reels.controller.ts:61` | `date` (one IST day), `dateFrom`, `dateTo` (ISO, `@IsDate`, `ReelsFiltersDto` :26–53); IST day bucketing done right (`reels.service.ts:286–300`); page/limit, no cap beyond pagination | **only `date`** (M:4100–4111; `dateFrom`/`dateTo` dropped) | no date control in the gallery (subject + status + sort only, `Reels.tsx:633–664`); `getReels` would send `date` (`api.ts:1411`) but nothing sets it; Refresh button exists | `reel_items`, permanent | **none on our route** (`ReelsFiltersDto` has no `exam`; the SME path has no exam at all; all 858 rows are `*` today) | custom range = proxy forwards `dateFrom`/`dateTo` + a UI range; exam needs a keyed reels list with `?exam=` (parity lane) |
| `/reels` views / watch time | `GET /reels/views?days=` M:4289 → **Mux Data** (out of scope) | n/a (Mux). Portal: `days` 1–90 (`Query(ge=1, le=90)`), Mux breakdown `limit=200` videos | — | always `days=30` (`getReelViews(days = 30)`, `api.ts:2742`; called with no arg, `Reels.tsx:1010`); no pills, no range | Mux Data retention (vendor) | none | out of scope (AI/Mux side). Note for the owner: with 858 reels and a 200-row breakdown, at most 200 reels can ever show a view count — a "no views" card may mean "outside Mux's top 200" |
| `/reels` blog map | `GET /sme/blogs` M:6244 → `sme-blog.controller.ts:37` | page, limit ≤100, `hasBlog`, `search` (`sme-blog.dto.ts:15–38`); no date | page/limit/hasBlog/search | page 1, limit 100 only | `reel_blogs`, permanent | none | not a windowed panel; fix = send `hasBlog=true` and page through, or look up per reel |
| `/simulations`, `/pyq`, `/mains`, `/psychometric` lists | keyed / keyless lists (§A.0) | **no date params** on any (`CmsPyqFiltersDto`, `CmsMainsFiltersDto`, `CmsPsychometricFiltersDto`, `listSimulations(exam)`) | — | exam selector (sims/pyq/mains), none (psychometric) | content tables, permanent | ✅ exam on sims/pyq/mains; none on psychometric | these are inventories, not time series — a window would only make sense as "created/updated between" (`createdAt`/`updatedAt` exist on every table) and would need new list params; recommend exam + language + Refresh only |
| Proposed usage clause on `/simulations` (§B) | would need a new keyed read | — | — | — | `simulation_attempts.startedAt/submittedAt`, permanent (plane A) | via the simulation's `examIds` | any window/custom range is computable without the 30-day `api_usage` cap |

## D. Contradictions

1. **Brief: `/content-edits` "rendered 0 rows."** Live `content_edits_snapshot.yml:157–180` shows Edits saved 1, one row (staff editor, 28 Oct 11:54, "Question text error", from a user report). Our audit has the matching PATCH at 10-28 11:54 IST (E8).
2. **Brief: `/reels` "rendered 0 rows."** Live `reels_snapshot.yml:134` "Reels Gallery (858)" with cards, "Edit blog" buttons and like buttons "0".
3. **Brief/§2.2: reels routes are "8".** True, but the portal also calls a 9th, `POST /reels/:id/like`, which does not exist (no `like` in `B:modules/reels`, controllers `reels.controller.ts`, `reels-user.controller.ts`).
4. **`content.api.ts:15–16` "the backend requires `_edit` and writes the content edit log from it"** — "backend" is the portal's FastAPI; our backend never receives `_edit` (popped M:8068) and keeps no person-level log.
5. **PYQ `duration`**: schema says minutes (schema.prisma:282), data says paper minutes (120/150), the portal says seconds with default 72, INGEST examples say 72. Four sources, two meanings.
6. **Mains "Marks"**: column header vs `tags[0]`; the editor hint "the first tag shows as Marks" is honest about the mechanism but wrong about the data (86 % of first tags are optional-paper names).
7. **Live mains editor copy "the marks field itself cannot be set"** (`mains_modal_edit_snapshot.yml:747`) vs `SmeUpdateMainsDto` (§5A.1 fact 6 — confirmed).
8. **`ingest-pyq`/INGEST §7 "Delete … keep using /cms"** vs the owner ruling 2026-11-03 (parity on `/sme/*`) — the doc needs rewriting when the lane lands.
9. **AUDIT §2.2 "`/cms/pyq` patch only as the before-snapshot"** — the before-snapshot is a GET (M:8078); the `/cms` PATCH is unused (C3 #10, reconfirmed).

## E. SQL run (prod, read-only, ids/counts only) and processes

| # | Query (shape) | Result |
|---|---|---|
| E0 | `information_schema.columns` for the content tables | `language` VARCHAR on pyq/mains/sims/reels/docs/psy test sets; none on `psychometric_questions` |
| E1 | `SELECT code, enabled FROM i18n_languages` | only `en` (enabled) |
| E2 | counts per bank × language × exam tag | pyq en/appsc 2,066, en/upsc 2,641; mains en/upsc 7,466; sims en/upsc 6; reels en/`*` 858; docs en/appsc 182, en/upsc 353; psy sets en 100; psy questions 2,000 (no lang). All active |
| E3 | mains `marks` × `tags[1]` | `marks`=10 on every row; first tags: 10 optional names (6,403), 15M 507, 10M 309, GS Paper 4 244, 12.5M 3 |
| E3b | year ranges | mains 2013–2025 (561 < 2014); pyq 1984–2026 (1,737 < 2014); 0 rows with year 0 |
| E4 | pyq by exam: withdrawn / no answer / E / optionE / paperLabel / image / verified | UPSC 0/0/0/0/0/0/0; APPSC 17/17/0/0/2,066/38/2,009 |
| E5 | pyq `duration` distribution | 120 ×3,214, 150 ×1,493 |
| E5b | pyq subjects by exam | UPSC CSAT 1,040; APPSC Mental Ability 275 (both absent from the subject filter) |
| E6 | orphans | `pyq_attempts` 57,396 total, 5,158 orphan on 2,962 ids (2026-04: 250, -06: 579, -07: 2,093, -08: 2,230, -09: 6); `mains_evaluations` 82, 0 orphan |
| E7 | psychometric profile | all `Onboarding`; difficulty EASY 1,000 / DIFFICULT 1,000; 0 null answers; 0 verified; 100 sets × 20 ids |
| E8 | `sme_audit_log` content/blog actions | CONTENT_INGEST PATCH 2,015 (2,009 on 10-01, then 10-22, 10-27 ×4, 10-28), POST pyq/bulk 23 (09-23→10-01); BLOG_CREATE 496, BLOG_UPDATE 496, BLOG_DELETE 1; 454 of 470 creates followed by an update < 10 s |
| E9 | simulations totalQuestions vs links | equal on all 6 |
| E9b | simulation usage, internal emails excluded | starters/finishers: Prediction 392/113, Sim-01 251/43, -02 21/10, -03 17/8, -04 29/13, -05 49/12 |
| E10 | mains `isOptional` | false 1,063 (all "GS Paper…"), true 6,403 |
| E11 | reels: blogs vs newest-100 by date; id format | 487 blogs, 100 in top-100, **387 beyond**; 858/858 UUID ids, all READY |
| E12 | content docs | 530 PUBLISHED, 5 DRAFT, 0 ARCHIVED; 0 duplicate `source_id`; 200 docs with questions recreated after creation (10,358 rows); `user_document_progress` vs attempts: 0 short rows (loss not measurable) |

Processes: only foreground `npx ts-node scripts/staging-db.ts --prod` runs, each exited; nothing left running, no pids to stop. No network calls besides prod SQL. No `git fetch` (portal HEAD 2026-10-27 matches the live v2.44.0 footer). Files written: this report only.
