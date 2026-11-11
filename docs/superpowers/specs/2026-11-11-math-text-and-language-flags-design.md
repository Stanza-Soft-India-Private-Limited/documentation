# Maths in question text + per-language module control — design

Status: APPROVED by the owner 2026-11-11. Transcribed (item B0) from the approved plan; the plan is the
source of truth and this file is its committed copy. The normative maths contract lives in
[`docs/contracts/math-text-v1.md`](../../contracts/math-text-v1.md) and its test vectors in
[`docs/contracts/math-vectors.json`](../../contracts/math-vectors.json) — the other three repos (mobile,
web, SME portal) copy the vectors file verbatim and link here.

B0 recon addendum (2026-11-11): two live-data hazards found while writing the vectors — a coding-decoding
option set using `$` as a symbol (`C%B$F$E` pairs into maths) and a stem with markdown-escaped
`\[…\]` brackets mid-sentence. The owner ruled on both the same day (contract §2.2 rule 2 and §6):

- **Ruling 21:** `\[…\]` display spans are LINE-ANCHORED — `\[` first non-whitespace on its line and the
  matching `\]` last non-whitespace on its line; a mid-sentence `\[…\]` is plain text. `$$…$$` and `\(…\)`
  are unchanged.
- **Ruling 22:** the coding-symbol option rows are fixed by a guarded SQL pass escaping `$` as `\$` before
  go-live (owner approves each statement at the time). The vectors keep documenting the pre-fix behaviour.

---

## Plan (as approved)

Date: 2026-11-11. Owner interview done (all rulings below are the owner's unless marked **my call**).
Method: `/orchestrating-serial-lanes` (I write no product code; one agent per repo lane, serial items, 5-min tick, I `git show` every diff). Staging deploys by the owner, migrations on staging by me, prod by the owner's hand.

### Context

PrepMonkey's prelims bank now carries CSAT (1,040 UPSC rows) and APPSC Mental Ability (275 rows); broader exams will bring real formulas (fractions, roots, limits, matrices). Today maths is faked with Unicode (`×`, `²`, `9³ + 9⁴ … 9¹⁰⁰`) or an uploaded figure (`questionImageUrl`). The owner wants formulas rendered as formulas on mobile (KMP/CMP), with the SME portal able to author them for new AND existing questions, and total backward compatibility for the 4,707 live prelims rows and the shipped 2.1/2.2 apps.

Second item (same plan, built after maths): per-exam feature flags exist (`exams.feature_flags`, 41 registry keys, `/config.features`, `@Feature` guard → 403 `FEATURE_DISABLED`). The owner needs the same switches per **language** (e.g. UPSC Hindi: mains not ready, explanations not ready) with English as the base.

#### What recon established (graph + prod read-only)
- Prod text has **no LaTeX** in stems/options; **129 explanations** and 2 option sets already contain `$…$` pairs (`$24,000$`, `$Q$`) written by the LLM explanation pipeline; currency like `$2 billion` also exists. 0 stems with `\frac`.
- Mobile has **three renderers**: stems → in-house `MarkdownText.kt` (bold/italic/escapes, keeps newlines, images split out by `extractInlineImages` → `StemWithFigures`); options → plain `Text`; explanation/approach/model answer/chat/docs → mikepenz `Markdown` (`ChatMarkdown.kt`, default GFM flavour, version 0.41.0). Research says the 0.41 GFM flavour treats `$…$` as math and **drops** it (fixed in 0.45.0) — unverified in-app, to be proven in M1.
- Web mirrors it: stems/options plain text, explanations `react-markdown` 10 (no math plugin). Portal: Lexical `MarkdownEditor`, no preview, `react-markdown` 9 available.
- Question text lives in: `pyq_papers` (mock tests reuse it), `mains_papers`, `simulation_questions`, `content_questions` (doc quizzes), `psychometric_questions` (OUT of scope).
- Native CMP maths libs: **huarangmeng/latex 1.7.2** (pure Kotlin, `inlineContent()` API, optional KaTeX font module) and **RaTeX-CMP 0.1.15** (Rust core via cinterop, ~99.5 % KaTeX syntax). Both released against **CMP 1.12.2**; we are on **CMP 1.11.1 / Kotlin 2.4.21 / AGP 8.13.2 / compileSdk 36**, and the 1.12 bump was shelved 2026-10-28 (forces AGP 9 + compileSdk 37). KaTeX-in-WebView (`compose-webview-multiplatform` 2.0.3, already a dep) works today but cannot self-size and has an open iOS LazyColumn bug.
- Backend: `X-App-Version`/`X-Platform` are sent by mobile since 1.7 and by web (`X-Platform: web`); the server only logs them, never compares. No `katex` dependency. `/config` already takes `@CurrentLanguage()` and calls `ExamsService.getEffectiveFlags(examId)`; `Vary: X-Lang` is set. Mobile already runs `SessionStateCleaner.refetchAfterLanguageSwitch()` on a content-language switch (`AppModule.kt:535`).

### Owner rulings (do not re-litigate)
1. **Notation:** LaTeX inline in the SAME text fields. No new columns.
2. **Delimiters:** pandoc-strict `$…$` (opening `$` followed by non-space; closing `$` preceded by non-space and NOT followed by a digit; no newline inside), `$$…$$` display (may span lines), plus `\(…\)` / `\[…\]` accepted. `\$` is a literal dollar. Same rule in all four codebases.
3. **Old clients:** server converts maths to readable plain text for clients below the maths-capable version (missing `X-App-Version` = old; web judged by its own version). Readable Unicode where safe (`x^2`→`x²`, `\frac{a}{b}`→`a/b` parenthesised when needed, `\sqrt{x}`→`√x`, `\times`→`×`, Greek, `≤ ≥ ≠`), else the LaTeX source.
4. **Scope:** prelims PYQ stem/options/explanation/approach (mobile + web) + simulations + mains + content-doc quizzes + Dify "Practice Similar" variations + AI chat answers. Psychometric OUT.
5. **Engine:** bounded spike on the CURRENT toolchain first (huarangmeng/latex, then RaTeX-CMP). If one links → use it behind a `MathRenderer` interface. If neither → STOP and the owner picks between WebView-KaTeX and reopening the CMP 1.12 upgrade.
6. **Consolidate** into one shared rich-text renderer per platform — **"make sure nothing is broken just because it's nice to consolidate"** → parity proof over every live prod row (see M1).
7. **`$`-drop bug** in mikepenz, if confirmed: fix in this build.
8. **Validation:** API rejects unparsable maths with a 400 naming field + bad span (strict KaTeX parse at save/import).
9. **AI prompts:** app-built Practice Similar + follow-up prompts get a "write maths as `$…$`" rule. Dify apps untouched.
10. **Portal:** ∑ formula button + cheat-sheet, live preview pane with the app's delimiter rules, save-time errors inline; Excel import lists bad rows before posting.
11. **Per-language flags:** one new jsonb column `exams.feature_flags_by_language` `{"hi": {"mains": false}}`, same registry keys, **narrowing only** (effective = exam flag AND language override), English = base with no column of its own. Learner UX identical to exam-off (hidden + 403). Flags only (caps/trial/pricing stay per exam). Portal = matrix (Default + one column per enabled language) with per-language readiness counts.
12. **Sequencing:** maths first, language control second, same plan.
13. **Tooling rule (owner, 2026-11-11):** for me and in EVERY agent brief: the knowledge graph first (`graphify explain/query/path` in each repo, cross-repo graph for cross-repo questions), Jev (`mcp__jev__jev_ask`) for triage/classification/report checks, `grep`/`rg` only for literal values the graph cannot hold (version strings, counts, exact header names). Say which tool answered what.

14. **Spike gate:** M0's outcome (screenshots, size delta, build log) is reported and I WAIT for the owner's go before M2 adopts an engine.
15. **Version:** maths-capable mobile release = **2.3 (versionCode 31)**; server floor `MATH_MIN_VERSION = { android: 2.3.0, ios: 2.3.0, web: 2.3.0 }` — web bumps `NEXT_PUBLIC_APP_VERSION` to 2.3.0 in the same deploy that ships its renderer. `min_app_version` (force update) is NOT raised.
16. **Paywall** generated free-vs-premium table stays exam-level in this build (follow-up noted).
17. **Validation allowlist:** the server AND the portal preview validate against the MOBILE ENGINE's supported command set (fixed after M0 from the library's published command list), on top of KaTeX-strict syntax. B1/B2/P2 finish after M0.
18. **Fallback is opt-in per learner-facing route** (`/pyq`, `/mains`, `/simulations`, `/content-docs`, mock, library, replay, weak-topics). Keyed `/sme/*`, legacy `/cms/*`, admin and script routes NEVER get rewritten (the portal proxy sends no `X-App-Version`; a rewritten edit form would save `a/b` over `\frac{a}{b}`). No consumers outside the four repos.
19. **Language for request-less jobs** (daily tasks, notification rules): `user_profiles.ui_language` — content language = UI language by the 2026-10-30 ruling. No new column.
20. **Validation scope:** only the fields an edit CHANGES are validated (an unrelated edit on a legacy row never 400s). New item B1b audits all prod text with the validator and reports; any cleanup (e.g. trivial `$5$` wrappers) is a guarded SQL pass the owner approves at the time.

### Calls I made (owner can veto while reading)
- Stems and options are NOT markdown-parsed beyond today's bold/italic/escapes (prod stems carry `1.`/`*` that a real markdown parser would mangle). So the shared renderer has two modes behind one API: **stem/option mode** (today's inline parser, newlines kept) and **markdown mode** (mikepenz full markdown for explanations/model answers/docs/chat). Maths and images use ONE pipeline in both modes.
- Daily tasks and notification rules that read flags get the request language when there is a request (`/tasks/today` already varies on `X-Lang`) and `user_profiles.ui_language` otherwise (ruling 19).
- Legacy `$…$` rows (129 explanations) are treated as the maths they were meant to be: 2.3 renders them in maths font, old clients get `24,000` instead of `$24,000$`. Both are improvements over today.
- 403 `FEATURE_DISABLED` body gains `language`. `enabled_modules` legacy mirror stays exam-level (old clients only).
- Chat context sent to Dify keeps raw LaTeX (LLMs read it); only human-facing previews/share text use the plain-text form.
- Content-doc BODY markdown gets maths for free through markdown mode; the portal's Lexical document editor gets no preview (only the four MarkdownEditor surfaces do). Noted as a freebie, not a lane.
- Spec document: after approval, item B0 writes `backend/docs/superpowers/specs/2026-11-11-math-text-and-language-flags-design.md` from this plan (the contract section verbatim) and commits it; the other repos link to it.

### The shared contract: "math-text v1"
One spec, four implementations (Kotlin, web TS, portal TS, backend TS), ONE vectors file copied verbatim into each repo and asserted by a unit test in each (`math-vectors.json`: `{input, segments[], plainText}` ×≥60 cases incl. currency, escaped `\$`, nested braces, `$$` across lines, `\(`, unbalanced, digits after closing `$`, Devanagari around maths, prod samples). I diff the four copies myself on every lane commit.
- `extractMathSegments(text) → [ {kind: text|inline|display, value} ]` runs BEFORE any markdown/inline parsing; maths spans never reach the markdown parser.
- `mathToPlainText(latex) → string` (ruling 3) — deterministic table; unknown command → source kept.
- `validateMath(text) → {ok} | {field?, span, message}` — backend only (KaTeX strict); portal/web pre-check with the same extractor for balance only.
- Render failure anywhere = show the source in monospace, never blank, never a crash.

### Build order and lanes
Producer before consumer. B = backend, M = mobile, W = web, P = portal (SME). Model per item by the `orchestrating-serial-lanes` rubric, written in each brief.

#### Lane B — backend (`~/Desktop/projects/backend`, main)
- **B0** spec file (above) + `docs/contracts/math-text-v1.md` + `math-vectors.json`. (sonnet)
- **B1** `src/common/math/`: `math-segments.ts`, `math-plain-text.ts`, `math-validate.ts` (add `katex` dep, `renderToString(..., {throwOnError:true, strict:'error'})` or `__parse`), vectors test. Also reject control char U+000C/`\f` in text fields with the hint "`\frac` inside hand-built JSON became a form feed" (the JSON-backslash trap). (opus)
- **B1b** Prod text audit (read-only via `staging-db.ts --prod`): run `validateMath` + the allowlist over every in-scope text field; report counts per field/exam and the failing spans; propose (not run) any guarded cleanup. (sonnet)
- **B2** Save-time validation (changed fields only, ruling 20) on every write route that stores question text: keyed `/sme/content/*` (pyq, mains, simulations, content-doc questions, bulk + Excel-fed bulk), legacy `/cms/*`, JWT `/pyq` `/mains` writes. Generated Prisma DTOs cannot carry a decorator → one `assertMathText(dto, fields)` helper called in the services; error `400 { code: 'INVALID_MATH', field, span, message }` whitelisted through `AllExceptionsFilter` like the 402/403 contracts. **`message` must itself read "`<field>`: `<reason>` near `<span>`"** because the portal's FastAPI proxy (`app/main.py` `_sme_proxy`, ~L4362) flattens upstream errors to `detail = body.message` and drops the other keys. Graph query lists the routes; brief must say "any write route the graph shows that the brief misses — report". (opus)
- **B3** Old-client fallback: `common/client-version.util.ts` (parse `X-App-Version` + `X-Platform`, absent → 0.0.0; tests) + `MathFallbackInterceptor` bound ONLY to the learner-facing controllers of ruling 18 (never `/sme/*`, `/cms/*`, admin, scripts): when client < floor for its platform, rewrite an ALLOWLIST of field names in the response body (`question, optionA..E, answerExplanation, solveTip, modelAnswer, questionText, content, text, title`… the agent lists every text field of the in-scope models from the schema) with `mathToPlainText`. **Fast path:** serialized body without `$`, `\(`, `\[` is returned untouched (byte-identity proven in a test with real prod-shaped bodies). Also apply `mathToPlainText` wherever the SERVER composes human text from those fields (push/share/notification templates — graph query `question` consumers). `verify-exam-backcompat.ts` gains a maths-free byte-diff probe at both old and new versions. (opus)
- **B4** Per-language flags: migration `exams.feature_flags_by_language jsonb NULL` (additive; verified 3× on a throwaway PG + `migrate diff` clean per the repo's own recipe — NOT the foreign `verify-migration` skill); `resolveFeatureFlags(overrides, languageOverrides?)` narrowing + parent-off propagation after the AND; `getEffectiveFlags(examId, language = 'en')` / `isFeatureEnabled(examId, key, language)` off the 60 s snapshot (snapshot carries the new column); `FeatureGuard` reads the request language; `/config`, `pyq.getContentTextFlags`, `mains.isMainsApproachEnabled`, `weak-topics.canViewPyqExplanation`, `tasks.resolveFlagsSafely` pass language; SME: `PATCH /sme/exams/:id { featureFlagsByLanguage: { hi: {key: false|true} } }` merge per language (true = remove override; `en` → 400; unknown language → 400 against `i18n_languages`; unknown key → 400 as today), GET detail returns raw + `effectiveFeatureFlagsByLanguage`; `GET /sme/exams/:id/content-counts?language=`. Invariants: no column/`en`/unknown language ⇒ byte-identical to today; UPSC untouched. (opus)
- **B5** Docs: `SME_CONTENT_INGEST_API.md` (maths contract + INVALID_MATH), `SME_EXAMS_API.md` (per-language matrix), `docs/staging.md` checklist for this wave; live-capture pass after the staging deploy. (sonnet, own worktree, rebased before each item)
- **B6** `scripts/pyq-ingest`: `build_rows.py` no longer rewrites `*`→`□` inside maths spans; `gate.py` validates maths (calls `node -e` with the same katex check); `ingest-pyq` skill note. (sonnet)

#### Lane M — mobile (`~/Desktop/projects/upsc_app`, main; spike in its own worktree)
- **M0 SPIKE (throwaway, ≤2 h hard stop, own `git worktree`):** on CMP 1.11.1 try, in this order (from the dependency lookup, Maven metadata read):
  1. **RaTeX-CMP `io.github.darriousliu:ratex:0.1.14`** — built exactly on CMP 1.11.1 / Kotlin 2.3.21; iOS = cinterop klib embedding a 9.85 MB `libratex_ffi.a` (no pods/SPM); Android AAR minCompileSdk 36, `.so` 2.3–3.5 MB per ABI; its POM pulls `material3 1.10.0-alpha05` → **pin material3 to ours** or the alpha wins resolution.
  2. **huarangmeng/latex `io.github.huarangmeng:latex-renderer:1.6.3`** (+ `latex-fonts-katex:1.6.3`) — pure Kotlin, built on CMP 1.10.3 / Kotlin 2.3.10 (Gradle upgrades its deps to 1.11.1); no release exists on 1.11; 1.7.x needs 1.12.2.
  Gate per candidate: `assembleDebug` + iOS simulator link (`linkPodDebugFrameworkIosArm64` / `compileKotlinIosSimulatorArm64`) + one screen: a sentence with inline `$\frac{a}{b}$`, a display `$$\sum_{i=1}^{n} i^2$$`, a Devanagari sentence with inline maths, dark theme; screenshots on both platforms; APK/IPA size delta; the library's published supported-command list captured for ruling 17. **Outcome reported to the owner, then WAIT (ruling 14).** Neither links ⇒ STOP (ruling 5). (opus)
- **M1** `core/richtext/`: `MathSegments.kt` + `MathPlainText.kt` + vectors test; the inline bold/italic/escape parser lifted out of `MarkdownText.kt` into a pure function (unchanged behaviour); `RichText(text, mode = Stem|Option|Markdown, figure…)` composable with a `MathRenderer` interface (no engine yet: a `SourceMathRenderer` that draws the source in monospace). **Parity proof:** read-only export of every live prod `pyq_papers`/`mains_papers`/`simulation_questions`/`content_questions` text field (content only, no user data) into a test fixture (gitignored, path in the brief); test asserts for every maths-free input `RichText`'s stem-mode AnnotatedString == old `parseMarkdown` output and option-mode == plain text; zero diffs or the item is not done. **`$`-drop proof:** unit test renders the 129 prod explanations through today's `ChatMarkdown` → if text is lost, fix: bump mikepenz to **0.44.0** (last release built on CMP 1.11.1; 0.45.0 was compiled on 1.12.0 and 0.46 declares foundation 1.12.1) and port the 0.45.0 fix (PR #627: render `INLINE_MATH`/`BLOCK_MATH` nodes verbatim in our own `markdownAnnotator`), since our extractor already removes real maths before the parser and only non-maths `$` (currency) reaches it — test proves all 129 render in full. (opus)
- **M2** Engine: `MathRenderer` implementation with the spike winner (`InlineTextContent` for inline, block for display); colour from `LocalContentColor`, size from the surrounding `TextStyle`; Hindi line-height rule (+2sp) respected; failure → source. (opus)
- **M3** Swap call sites to `RichText` (graph: 17 files use `MarkdownText`/`StemWithFigures`/`ChatMarkdown`): stems (PracticeComponents, PyqQuestionDetailScreen, PracticeQuestionScreen, MockTestScreen, MockTestQuestionReviewScreen, SimulationTestScreen, SimulationReview*), options (PyqQuestionDetailScreen, EnhancedQuestionDetailsModal `OptionsSection`, mock/sim option rows), explanation/approach/model answer (PracticeComponents `ExplanationView`, PyqComponents, MockTestQuestionReviewScreen, MainsFlowScreens, MainsEvaluationTab), chat (`ChatScreen` via `ChatMarkdown` → markdown mode), content docs. List previews (`PracticeListScreen`, `MockTestResultScreen`, replay) use `mathToPlainText` on the one-line preview. Old renderers deleted or reduced to aliases; `extractInlineImages` reused unchanged. (opus)
- **M4** Prompts + decode: `buildQuestionContextMessage`/variation prompt in `PrelimsViewModel` + web parity text get the rule "write every formula as `$…$` / `$$…$$`"; `VariationQuestionLenientDecodeTest` gains a `\frac` case (backslashes survive the Dify JSON path, including fenced JSON). (sonnet)
- **M5** Per-language flags: no new plumbing (config refetch on language switch exists). Add a test that a `/config.features` change after a language switch hides/returns the Mains tab, and that the timed-screen hold defers it like content does. (sonnet)
- **M6** Release prep: `release: 2.3 (versionCode 31)`, i18n keys for any new strings (`i18nCheck -PrunTests`, KeyParityTest, LiteralLeakTest, NegativeTrackingTest all green, counted from a full run), release notes per `reference_release_notes`. (sonnet)
- **M7** Device drivers on staging (after the staging deploy + seed): Android emulator (skill `prepmonkey-android-rig`) + iOS simulator: maths rows in English and Hindi across prelims/mock/simulation/mains/doc quiz, light+dark, long-list scroll, Practice Similar on a CSAT maths question, Hindi `mains:false` → tab gone → back to English → tab back. Screenshots judged by me (patched build, FLAG_SECURE off only in the driver copy). (opus)

#### Lane W — web (`~/Desktop/projects/prepmonkey-web`, main; skill `prepmonkey-web-rig`)
- **W1** `src/lib/richtext/`: extractor + plain-text + vectors test (same JSON). (sonnet)
- **W2** `RichText` React component: stem/option mode (whitespace-pre + today's inline rules) and markdown mode (`react-markdown` + our pre-extracted maths, `katex` CSS + `katex.renderToString`, NOT remark-math's single-dollar rule which is not pandoc-strict); failure → source. (opus)
- **W3** Swap sites: `dashboard/practice/page.tsx` (stem, options, explanation, follow-up chat, variation), `MockTestFlow.tsx`, `SavedQuestionsList.tsx`, `practice/mains/page.tsx`, simulation + doc quiz components; previews via plain-text; `NEXT_PUBLIC_APP_VERSION` → 2.3.0; Playwright driver from a `git archive` against staging with every write stubbed (keepalive stripped per `feedback_playwright_keepalive_stubs`). (opus)

#### Lane P — SME portal (`~/Desktop/projects/upsc-document-generator`, skill `sme-portal` FIRST; worktree per lane, gate from `git archive`, ff-merge = prod deploy)
- **P1** `admin-ui/src/lib/richtext/` extractor + vectors; `RichTextPreview` (stem mode + markdown mode, KaTeX). (sonnet)
- **P2** `MarkdownEditor`: ∑ button inserting `$…$`, cheat-sheet popover, live preview pane (toggle, remembers state), used on PYQ / Mains / Simulations / doc quiz block; `INVALID_MATH` 400 shown beside the field; `ExcelImportDialog` pre-check (balance) + server row errors listed before posting; FastAPI proxy: `_sme_proxy` forwards `json_body` verbatim (verified) but flattens error bodies to `message` — either extend it to pass `code/field/span` through (preferred, with a pytest under the socket guard) or rely on the message text from B2. Doc quizzes: `QuizBlockNode.tsx` `parseQuizText` parses the raw quiz text line by line — it must tolerate `$`/`\` in question, option and explanation lines, and the block's read view renders through `RichTextPreview`. Gate: tsc, vite build, py_compile, pytest with the socket guard. Browser check on a local Vite serve against STAGING, never prod writes. (opus)
- **P3** Exams → Features tab matrix: Default column + one per enabled language (`useContentLanguages`), language cell disabled when the exam-level flag is off, pending-state preview honest (parent-off AND language), save → `featureFlagsByLanguage`; readiness counts under each language header from `content-counts?language=`; docs ledger + `/exams` "Linked" panel. Requires B4 COMMITTED and on staging. (opus)

#### Order across lanes
B0 → {M0 spike ∥ W1 ∥ P1 ∥ B1 syntax half} → owner's go on the engine (ruling 14) → allowlist fixed → B1 allowlist + B1b, B2, B3 → {M1 → M2 → M3 → M4, W2 → W3, P2} → B4 → {P3, M5} → B5, B6, M6 → owner deploys STAGING (backend image + I run `staging-prisma.ts migrate deploy/status/drift`) → staging seed (≈20 CSAT-style maths rows en + hi via the SME API, "[STAGING TEST]" prefix; `hi: {mains:false}` on UPSC staging) → B5 live capture, M7, W3 driver, P2/P3 browser checks → report → owner: prod backend deploy + migration (ingest freeze not needed: migration is additive) → web push (auto-deploy) → portal release (ff-merge) → mobile 2.3 store build by the owner.

### Verification (end to end)
- Unit: vectors test green in all four repos and the four JSON copies byte-identical; mobile parity test 0 diffs over every live prod text; `$`-drop test 129/129; backend byte-identity test for maths-free bodies at old and new versions; full suites (backend 141/141 jest, mobile ~700 tests counted from a full `--rerun` run, web vitest, portal pytest with socket guard).
- Staging API probes (15/15): `POST /sme/content/pyq` with bad maths → 400 INVALID_MATH with field+span; good maths → 201; `GET /pyq/:id/details` with `X-App-Version: 2.2.0` → plain text, `2.3.0` → LaTeX, no header → plain text, `X-Platform: web` + 2.3.0 → LaTeX; a maths-free row byte-identical across all four; `/config` with `X-Lang: hi` after `hi:{mains:false}` → `features.mains=false`, without header unchanged; `GET /mains/filters` with `X-Lang: hi` → 403 FEATURE_DISABLED carrying `language`; `migrate status` clean, `drift` none.
- Devices: M7 above; web Playwright; portal browser checks (preview pane renders the same bytes the app shows for 5 chosen rows — screenshot side by side).
- Prod after rollout: the same probes read-only; UPSC content-counts unchanged; `verify-exam-backcompat.ts` against prod.

### Reminders the owner asked for ("what am I missing")
1. **JSON backslash trap:** `\frac` in hand-built JSON (Excel→JSON scripts, LLM-emitted JSON) becomes `\f` (form feed) + `rac`. B1 rejects U+000C with a hint; ingest scripts use `json.dumps` (safe).
2. **Accessibility:** maths `contentDescription`/aria-label = `mathToPlainText` output.
3. **Dark mode, font size, Hindi tracking:** maths takes the surrounding colour/size; the negative-letterSpacing Latin-only rule and +2sp Hindi line height must still hold with inline maths (M2 + M7).
4. **App size:** fonts/static lib measured in M0; reported before M2.
5. **Portal must ship BEFORE SMEs author maths** (preview + inline errors), and the backend fallback must be LIVE before any maths row goes to prod — the rollout order above enforces it.
6. **Search** on `/pyq` matches raw text (`\frac` is searchable, `½` is not) — accepted, noted in docs.
7. **Hindi + maths rows** on staging for the drivers. (Prod Hindi content was 0 rows when this was written; since 2026-11-11 evening UPSC prelims carries 1,100 `hi` rows with `prelims.explanation` off for `hi`.)
8. **Psychometric** excluded; **content-doc body** gets maths free of charge, Lexical doc editor has no preview (freebie, documented).
9. **Per-language flags are a ceiling, not a bridge:** the folder-level English fallback already shows English rows when a Hindi folder is empty; the new flag is for deliberately hiding a module (ruling 11).
10. **Legacy `enabled_modules`** stays exam-level; 1.8/1.9 clients never see language flags.
11. **Memory + graphs:** `graph-sync.sh` + `--update` per repo at close, cross-repo re-merge; memory files for the maths contract and language flags; worklog entry (portal items in `SME_WORKLOG.md`, never `WORKLOG.md`).
12. **Docs mirror:** backend `docs/**` is auto-mirrored to a PUBLIC GitHub repo — no PII/revenue in the new docs.
13. **Never prod from me:** staging migrations mine, prod deploys + migrations the owner's; `mau` ships the working tree, so lanes commit everything before any deploy.

### Addendum B1g (2026-11-11) — ruling 17 as the engine's documented support
The B1b prod audit showed the scored-suite allowlist rejecting standard KaTeX names the engine renders (`\tan` 28×, `\tau` 6×, `\max`, `\sup`) and strict mode rejecting Unicode symbols inside maths (`–`, `”`, `₹`; 11 fields). Orchestrator decision: stage 3 rejects only `MATH_ALLOWLIST_EXCLUDED` (macros, links/images/html, README-unsupported); the generated suite list stays as DATA for non-blocking `warnings` (logged at debug on writes, no response shape change); stage 2 ignores the strict codes `unknownSymbol`/`symbolNotInFont` (the engine's Unicode font fallback draws them). Genuine syntax errors still fail. Contract §4 updated; public signatures unchanged (other repos copy only segments/plain-text).

### Addendum: Built (backend, e46727c..e5fa694)
- `e46727c` B0 — math-text v1 contract, hand-computed vectors, this design spec.
- `2b3b743` B1 syntax half — segment extractor, plain-text conversion, KaTeX-strict validator.
- `3987a30` B1c — rulings 21/22: `\[…\]` line-anchored; hazards and authoring rules in the contract.
- `10fc907` B1d — freeze the clock in the countSegment parity test (was flaky).
- `6c25535` B1e — engine allowlist (RaTeX-CMP 0.1.14) as validation stage 3.
- `49c325c` B2 — save-time maths validation on every question-text write, changed fields only.
- `7a2c33f` B3 — opt-in old-client maths fallback, client-version util, back-compat probe.
- `0efb465` B1b — read-only question-text maths audit script.
- `d375660` B1f — table lookups are own-property only.
- `c299fb1` B1g — stage 3 rejects only engine exclusions; Unicode symbols pass stage 2.
- `e5fa694` B4 — per-language feature flags, narrowing only (migration `20261111000000_feature_flags_by_language`).
- B5 (docs) and B6 (ingest scripts) follow this addendum.
