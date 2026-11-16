# SME Activity Path Rules API — the learner's walking path

> New 2026-11-16 (backend items B1, B1b, B2, B8). Committed, not deployed.

Changing the rules of the learner's **activity path** for an exam without a deploy. The path is
a Duolingo-style walk of **steps**, grouped into **camps** inside **stages**. Each step has 1–6
measurable **goals**. The server computes progress from data the app already records and
permanently records what each learner earned.

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Auth:** `x-api-key: <API_KEY_SECRET>` on every `/sme/*` request.

> **There is no response envelope.** Success bodies are raw; errors are shaped by the
> exception filter. Branch on the HTTP status code, never on a `success` field.

> **Invoke the `frontend-design` skill before building any screen from this doc.**

---

## 1. The whole mental model, in one page

**The rules live in two places and are merged.**

| | |
|---|---|
| **Code defaults** | A complete rule set shipped with the backend. **Only `upsc-cse` has one** (stage "The Ascent" = Prelims, 6 camps × 5 steps; stage "The Summit" = Mains, 3 camps × 5 steps). |
| **Override** | `exams.activity_path_rules`: a **partial** rule set stored per exam, merged over the defaults. Usually `null`. This is what you edit. |
| **Effective** | defaults + override = what learners are **served**. An exam with no defaults gets a path only if its override is a **complete, valid** rule set on its own. |

**An invalid stored override is ignored, never half-applied.** If the stored blob fails
validation (for example after a later backend release tightened a rule), learners get the
code defaults and `GET` shows you the errors, path by path.

**The on/off switch is not here.** It is the ordinary feature flag `dashboard.activityPath`
on `PATCH /sme/exams/:id` (§7). Migration `20261116000000` seeded it **off** for every exam.

**Hindi wording (rules version 9).** Write `hi` text with the app's words, so a screen never
mixes two terms: a Phase is **फ़ेज़** (चरण means a Step only), the journal is **डायरी**
(feminine: "आपकी डायरी"), questions are **सवाल**, and the names **Monk** and **Streak** stay in
Latin letters. Since version 10 also: the learner's Journey is **सफ़र** (masculine: "आपका
सफ़र"), Share is **शेयर करें**, and a journal entry is a **नोट** (not प्रविष्टि). An override
saved earlier keeps its own wording until you edit it.

### What a rule set contains

| section | what it is |
|---|---|
| `version` | integer, bumped when the rules change meaningfully; echoed to the app |
| `prelims.resultDate` | `YYYY-MM-DD` or null — the day the Prelims result is out: from then the "Did you clear Prelims?" prompt appears (rules version 7, see §9). **Set the new date every year** |
| `prelims.text` | rules version 7, optional: the Prelims-flow copy (39 keys since version 8, en + hi, merged **by key**), see §9 |
| `calibrationSimulationId` | a simulation id or null — the calibration test offered after "did not clear" (§9) |
| `subjects[]` | `{key, label{en,hi}, pyqSubject, roadmapSubjects[], mockSubjects?[]}`: how a path subject maps onto the PYQ spelling (accuracy), onto study-material folders (topics) and, since rules version 6, onto mock-test subjects (the confidence judge's two mock signals). `roadmapSubjects: []` = no study material: no topic goals possible, stars from questions only. `mockSubjects` absent / `[]` = no mock mapping: the mock signals are excluded for that subject |
| `stages[]` → `camps[]` → `steps[]` → `goals[]` | the path. Keys must be unique; a goal key is a learner's permanent record, so **never reuse a key for a different goal** |
| `stars.levels` | exactly three per-subject levels: covered / solid / mastered (see **Star levels** below) |
| `titles[]` | earned titles and when they unlock (see **Title conditions** in §6) |
| `pillars` | the four score pillars (weights sum to 100; each pillar's inputs sum to 100) — see **Score pillars** below |
| `topics` | `{weakBelowPercent, minAnswers}`: when a topic card on the score screen gets the "Weak" tag (rules version 5) — see **Topics** below |
| `confidenceJudge` | thresholds, the six signal weights (sum to 100) and, since rules version 6, the banner text — see **Confidence judge** below |
| `notes` | Monk's journal templates per situation, `{key, en, hi}`; `default` must have at least one (see **Monk's journal notes** in §6) |
| `stageNotes` | rules version 8, optional: `{minDaysOnPath, minSyllabusPercent}` (defaults `30`, `50`) — from which number a `stage_cleared` / `stage_restart` note may name the learner's days on the path / syllabus coverage (see **Monk's journal notes** in §6). `minDaysOnPath` is a whole number ≥ 2, `minSyllabusPercent` 0–100; when sent in a complete rule set, both |

**Goal types:** `subject_accuracy` {subject \| slot, minPercent, minAnswers} · `topics_completed`
{subject \| slot, count} · `active_days` {days, withinDays?} · `mock_score` {minPercent} ·
`questions_attempted` {count} · `confidence_gap_cleared` {subject \| slot} · `mains_evaluated`
{count} · `streak_days` {days}. `streak_days` is **available, but the defaults do not use it**
(owner ruling 2026-11-16: the path must not depend on the streak until the streak system is
reviewed).

`active_days` counts **distinct IST days on which the learner studied in this exam**: answered a
PYQ, answered a library MCQ, read or completed a document, submitted a mock, or had a Mains answer
evaluated. `withinDays` absent or null means lifetime; `N` means within the last N days, today
included, and must be at least `days`. Title placeholders: `{days}`, plus `{withinDays}` when it is set. Every goal has `key`, `type`, `title{en,hi}` with `{placeholders}`, `link` (must start
with `/open/`, see [DEEP_LINKS.md](./DEEP_LINKS.md)) and optionally `lockedUntilOthersDone: true`
(at most one per step, and it must be the last goal). A `slot` (`weakest_subject`,
`second_weakest_subject`, `least_covered_subject`) is resolved per learner and then frozen.

**Star levels** (rules version 3): each level is
`{level, key, label{en,hi}, description?{en,hi}, minCoveragePercent, minAccuracyPercent, minAnswers, requireNoConfidenceGap, minReviewDays?, minMockPercent?}`.
A learner earns a level when every criterion that applies to the subject holds:
- `minCoveragePercent`: % of the subject's study material completed. Skipped for a subject with
  no study material, so that subject earns stars from questions only. 0 means not required.
- `minAccuracyPercent` and `minAnswers`: PYQ + library accuracy over at least that many answers.
  Skipped for a subject with no PYQ spelling.
- `requireNoConfidenceGap`: the confidence judge must say "clear".
- `minReviewDays`: number of distinct days on which the learner re-answered a question of the
  subject that was first answered on an earlier day. Absent or 0 means not required.
- `minMockPercent`: best submitted mock score in the exam. It is exam-wide, not per subject.
  Absent or null means not required. It is skipped while the exam has no active mock.

Levels are climbed in order and **a star is never taken back**, even if accuracy later drops or
the learner restarts the path. Rules that apply to every level:
- Thresholds may not go down from one level to the next. For example, Mastered cannot ask for
  fewer review days than Solid.
- `description` is the card text on the app's "Your subject stars" sheet. It may use
  `{minCoveragePercent}`, `{minAccuracyPercent}`, `{minAnswers}`, `{minReviewDays}` and
  `{minMockPercent}`. Use `{minMockPercent}` only on a level that sets it.
- A level without `description` shows text generated from its numbers. This is how levels from
  an override written before version 3 still validate.
- Unknown fields on a level are rejected, so a typo such as `minReviewDay` fails validation.

Because `stars.levels` is replaced whole, a PATCH must send all three levels. Defaults: Solid
needs 1 review day; Mastered needs 3 review days and a 50% mock. The owner has not confirmed
these numbers yet.

**Score pillars** (`GET /activity-path/score`, rules version 5): each pillar is
`{label{en,hi}, explanation?{en,hi}, weight, inputs[{signal, weight, target?}]}`. The app shows
`label`, the percent and `explanation` (one line under the bar, **no placeholders**). Absent
`explanation` means a line generated from the inputs, so an override written before version 5
still validates. If you change a `target`, also change the explanation text that quotes it
("…in the last 30 days"). A signal with no data is left out of its pillar, and the other weights
are scaled up to fill the gap. A pillar with no data at all shows no percent (null, not 0). The
overall score is the weight-weighted mean of the pillars that have a percent.

| signal | 0–100 means | `target` | no data when |
|---|---|---|---|
| `topic_coverage` | study material completed over every mapped subject (partial reading counts) | — (rejected) | the learner has no active day, or no mapped documents exist |
| `questions_attempted` | distinct questions answered ÷ target | **required**: the count that reads 100% | no active day |
| `retry_accuracy` | accuracy on questions re-answered on a later day (latest answer) | — | nothing re-answered on a later day |
| `recency` | per touched subject, 100 if practised today down to 0 at `target` days ago; averaged | **required**: days until fully stale | no subject ever touched |
| `active_days_ratio` | study days in the last `target` days ÷ `target` (**not** the streak) | **required**: the window in days | no active day |
| `streak` | live streak ÷ target. **Do not use** until the streak system is reviewed (owner ruling) | **required** | no active day |
| `pyq_accuracy` | PYQ accuracy, latest answer per question, every subject | — | no PYQ answered |
| `mock_score` | **mean** accuracy of every submitted mock in the exam | — | no mock submitted |
| `mains_evaluation_score` | mean of score ÷ marks over the latest evaluation per Mains question | — | no Mains evaluation |

Defaults: Preparation 30 (`topic_coverage` 60 + `questions_attempted` 40, target 3000) ·
Retention 20 (`retry_accuracy` 60 + `recency` 40, target 14) · Consistency 25
(`active_days_ratio` 100, target 30) · Performance 25 (`pyq_accuracy` 50 + `mock_score` 50).
The owner has not confirmed these numbers yet.

**Topics** (rules version 5): `{"weakBelowPercent": 40, "minAnswers": 5}`. A topic card gets the
"Weak" tag when its accuracy is **strictly below** `weakBelowPercent` over **at least**
`minAnswers` answers. `weakBelowPercent` is a number greater than 0 and at most 100;
`minAnswers` is a whole number greater than 0; any other field is rejected. Merged **by field**:
`{"topics": {"minAnswers": 8}}` keeps the stored `weakBelowPercent`. A topic is a study document
for a subject with study material, and a PYQ topic otherwise. PYQ topics have only about 1–3
questions each, so few of them reach 5 answers. The owner has not confirmed these numbers yet.

**Confidence judge** (rules version 6, judge v1). The app shows no confidence chips. The backend
infers "confidence is way higher than accuracy" from behaviour and shows a banner with a "Take
reality check quiz" button on a subject or study document when the judge says **gap**. It uses
`{minAnswers, gapAtScore, clearedAtScore, signals{…}, banner?{message{en,hi}, actionLabel{en,hi}}}`:

| signal | value | threshold field (default) | weight | left out (no data) when |
|---|---|---|---|---|
| `coverageAccuracyGap` | coverage % − accuracy %, in points | `minGapPercent` (25) | 25 | no study material |
| `fastAndWrong` | wrong PYQ answers given in under `fastSeconds` (15) ÷ wrong PYQ answers with a measured time | `minSharePercent` (20) | 20 | no answer with a measured time yet (apps start sending times with the next build) |
| `timedVsPractice` | practice accuracy − accuracy on the subject's mock questions, in points | `minGapPercent` (15); `minMockAnswers` (10) | 15 | no `mockSubjects`, or fewer than `minMockAnswers` answered mock questions |
| `skipPattern` | unanswered ÷ presented mock questions of the subject | `minSkipPercent` (20); `minMockAnswers` (10) | 10 | no `mockSubjects`, or too few mock questions |
| `wrongOnRetry` | re-attempted PYQs still wrong on the latest attempt | `minSharePercent` (30) | 20 | nothing re-attempted |
| `staleness` | days since the subject was last touched | `staleDays` (21) | 10 | never touched |

- **Score:** fired weight ÷ weight of the signals with data × 100. A weight of 0 switches a
  signal off.
- **Floor:** below `minAnswers` (20) answers in the subject, the state is `unknown`.
- **Evidence:** coverage gap, fast-and-wrong, timed-vs-practice and wrong-on-retry are the
  evidence; skip pattern and staleness are modifiers. With no evidence signal holding data, the state is `unknown`.
- **States:** `gap` when score ≥ `gapAtScore` (50) and an evidence signal fired, else `clear`.
  Goals and the Mastered star need **cleared**: `clear` and score ≤ `clearedAtScore` (30).
- **Optional fields:** `banner`, `minMockAnswers` and `mockSubjects` are optional. An override
  written before version 6 stays valid: the built-in banner words and a mock floor of 10 apply.
- **Merging:** `banner` is merged by field like the other `confidenceJudge` scalars, and each
  signal is replaced whole.
- **Owner to confirm:** the weights and thresholds are still the B1 numbers.

**See why the judge decided something:** `GET /sme/activity-path/judge/:examId/:userId` (same
`x-api-key`, **read-only**: it writes nothing and earns the learner nothing). For every subject
it returns:
- the inputs it read: answers, accuracy, coverage, re-attempts, timed and fast-wrong answers, mock
  tallies, last touch;
- each signal's value, threshold, weight, whether it had data and whether it fired;
- the score, the state, `cleared`, and the reason when `unknown` (`below_floor` | `no_evidence`).

Unknown user, or an exam without rules → `404`. Use it with real learners before changing weights.

**Text:** `en` and `hi` must use the **same placeholders** (the validator rejects a
translation that drops `{minPercent}`).

---

## 2. Read — `GET /sme/activity-path/rules/:examId`

```json
{
  "examId": "upsc-cse",
  "hasDefaults": true,
  "defaults": { "...the code defaults...": "" },
  "override": null,
  "effective": { "...what learners are served...": "" },
  "overrideApplied": false,
  "rulesVersion": 4,
  "errors": [],
  "cacheNote": "Rules are cached for 60 s per replica: …"
}
```

| field | meaning |
|---|---|
| `override` | the **stored** partial, exactly as stored (null = none) |
| `effective` | what learners get. `null` = this exam has no path |
| `overrideApplied` | `false` with a non-null `override` means **the override is being ignored** |
| `errors` | `[{path, message}]` explaining why the stored override is ignored. Empty when applied or absent |

`404` for an unknown exam.

---

## 3. Change — `PATCH /sme/activity-path/rules/:examId`

```json
{ "override": { "...a PARTIAL rule set...": "" }, "dryRun": false }
```

- **`override` is merged INTO the stored override.** It never replaces it wholesale: a key you
  do not send keeps its stored value.
- **`override: null` clears everything** back to the code defaults. It is the only way to remove
  the whole override, and it is audited like any other change.
- `override` is required. Leaving it out is a `400`, so "no change" and "clear" never look alike.
- **Any other top-level field is a `400`.** Nested rule keys are passed through to the rule validator.

### What a rule edit does to learners who already earned something (read before editing)

A rule edit is **not** a restart. Two rules, applied everywhere:

1. **What was earned stays earned.** Goals, steps, camp stamps, stars, titles and journal notes
   are kept: they were awarded under the rules of that day. "My stats" keeps every stamp and the
   title, a stamped camp still counts for camp/stage titles, and notes are never rewritten.
2. **The path and its celebrations follow the rules in force now.**
   - **Path:** a camp shows "cleared" only while all of its current steps are done. Make a
     cleared camp longer and learners see it as their current camp again; "My stats" counts camps
     the same way.
   - **Camp celebration:** plays only when the camp is in the learner's current stage and
     cleared under the current rules.
   - **Star celebration:** plays only when its subject and level still exist.
   - **Title celebration:** plays only when the title still exists. A camp/stage title also needs
     that camp/stage cleared, when it is in the learner's current stage.
   - **Withheld celebrations are kept, not thrown away.** They play the moment the learner
     genuinely reaches that state under the new rules.

Example:
1. You shrink Base Camp to one step, and a learner clears it.
2. You undo the shrink. The learner is back on Base Camp step 2. Nothing plays "Base Camp
   reached" or "The Trailblazer", but both stay in their record.
3. The celebrations play when they clear Base Camp under the restored rules.

**Practical advice:** test a shortened camp with `dryRun: true`, not on live learners.

### Merge semantics (exactly what the server does)

| part | granularity |
|---|---|
| `version`, `calibrationSimulationId`, `prelims.resultDate` | replaced by field |
| `prelims.text` | **by key** (send one line, the other keys stay) |
| `stages` → `camps` | **by key**: a stage or camp you send is patched field by field (`label`, `title`, `stamp`, …); its child list merges by key too |
| `steps`, `subjects`, `titles` | **by key, replaced WHOLE**: send the complete item. A step's goals are one unit, because their order and the locked-last rule only make sense together |
| new keys | appended in the order sent |
| `{ "key": "...", "remove": true }` | deletes that item. Removing a key that exists nowhere is a `400`: a typo must not silently do nothing |
| `stars.levels` | replaced whole (the three levels only validate together) |
| `pillars`, `confidenceJudge.signals`, `notes` | per entry: per pillar, per signal, per situation (a situation's template **list** is replaced whole) |
| `confidenceJudge` scalars | by field |
| `topics` | by field |
| `stageNotes` | by field |

### Validation: three checks, all before anything is written

1. the **incoming** partial is shaped like an override;
2. the **resulting stored** override is shaped like one;
3. the **resulting effective** rules are a complete, coherent rule set.

Any failure is a `400` and **nothing is written**:

```json
{
  "message": [
    "resulting effective rules › stages[0].camps[0].steps[1].goals[1].minPercent: must be a number from 0 to 100"
  ],
  "error": "INVALID_ACTIVITY_PATH_RULES"
}
```

`message` holds **every** problem as `<check> › <path>: <message>`, so the portal can show them all at once.

If the stored override is not even shaped like an override, a merge into it is refused (`400`,
check `stored override (invalid shape …)`). Clear it with `{ "override": null }`, then re-apply.

### Response

The same body as `GET`, plus `dryRun` and `changed`. `changed: false` means the PATCH left the
stored override as it was: nothing was written and nothing audited. Sending the same PATCH twice
is therefore harmless.

### `dryRun: true`

Runs the same three checks and returns what `override` / `effective` **would** be. Nothing is
written, audited or invalidated. Use it for the portal's "preview" button.

### Audit and caches

Every real change writes an `sme_audit_log` row: action `EXAM_UPDATE`, `actorNote`
`activity_path_rules updated` / `activity_path_rules cleared`, and `before` / `after` of the
column. The resolver caches rules for **60 s per server replica**. The replica that took the
PATCH serves the change at once; **other replicas converge within 60 seconds**.

---

## 4. Worked example — change one goal's threshold

"Step 2 of Base Camp: make the Polity accuracy goal 35% instead of 30%." Steps are replaced
whole, so send the **complete** step with the one number changed (copy it from `effective`):

```json
{
  "override": {
    "stages": [{ "key": "ascent", "camps": [{ "key": "ascent.c1", "steps": [{
      "key": "ascent.c1.s2",
      "title": { "en": "Step 2", "hi": "चरण 2" },
      "goals": [
        { "key": "ascent.c1.s2.attempt", "type": "questions_attempted", "count": 25,
          "title": { "en": "Attempt {count} questions", "hi": "{count} सवाल हल करें" }, "link": "/open/pyq" },
        { "key": "ascent.c1.s2.accuracy-polity", "type": "subject_accuracy", "subject": "polity",
          "minPercent": 35, "minAnswers": 10,
          "title": { "en": "Achieve {minPercent}% PYQ accuracy in {subject} ({minAnswers}+ questions)",
                     "hi": "{subject} में {minPercent}% PYQ सटीकता हासिल करें ({minAnswers}+ सवाल)" },
          "link": "/open/practice_list?filterType=subject&filterValue={pyqSubject}&examType=prelims" },
        { "key": "ascent.c1.s2.active", "type": "active_days", "days": 3,
          "title": { "en": "Study on {days} different days", "hi": "{days} अलग-अलग दिन पढ़ाई करें" },
          "link": "/open/pyq" }
      ]
    }] }] }]
  }
}
```

The other five camps, the other four steps and the Summit stage are untouched. Learners who
already earned this goal keep it: earned goals are permanent.

## 5. Worked example — map a subject to study material

"Governance & Society questions should count Indian Society documents as its topics."

```json
{
  "override": {
    "subjects": [{
      "key": "governance_society",
      "label": { "en": "Governance & Society", "hi": "शासन एवं समाज" },
      "pyqSubject": "Governance & Society",
      "roadmapSubjects": ["Indian Society"]
    }]
  }
}
```

Run on a throwaway database on 2026-11-16: the stored column went from `NULL` to exactly this
`subjects` list, the served rules immediately listed `["Indian Society"]`, and the same PATCH
sent again returned `changed: false` (no write, no second audit row).

A study-material folder may belong to **one** path subject only; mapping it twice is a `400`. A
subject may own several folders (`"ir_security"` owns `"International Relations"` and
`"Internal Security"`). Links open the **first** folder.

## 6. Worked example — add a journal note template

A situation's template list is replaced **whole**, so send the existing templates plus the new
one (copy them from `effective.notes.mock_completed`):

```json
{
  "override": {
    "notes": {
      "mock_completed": [
        { "key": "mock_completed.1", "en": "…existing…", "hi": "…मौजूदा…" },
        { "key": "mock_completed.2", "en": "…existing…", "hi": "…मौजूदा…" },
        { "key": "mock_completed.3",
          "en": "{percent}% in today's mock. Note the three topics that cost you most.",
          "hi": "आज के मॉक में {percent}%। जिन तीन टॉपिक में सबसे ज़्यादा अंक गए, उन्हें लिख लीजिए।" }
      ]
    }
  }
}
```

Note keys are unique across all situations. A template may use only the placeholders its
situation provides (table below), and `en` and `hi` must use the same ones.

### Monk's journal notes

There is no AI in the journal. Each note is one of these templates, filled with the learner's real
numbers for that IST day.
- **When a note is written:** one note per **active** IST day, stored once with both languages
  rendered. A note is never changed after that, so a template edit only affects future notes.
- **Choosing the situation:** the first situation that matches, in the order below, is used. If none
  of its templates can be filled that day, the next situation is tried.
- **Choosing the template:** the variant is the day number mod the number of templates that fit, so
  consecutive days rotate.
- **History before the first open:** the days before the learner first opened the path get exactly
  one `story_so_far` note. `{dayNumber}` works in every situation.

Rules version 4 removed `streak_kept` and `streak_broken` (owner ruling: the path does not depend
on the streak). It also removed `confidence_gap`, because a past day's coverage cannot be rebuilt,
and `exam_near`, because the path does not read the exam date. An override that still uses them is
rejected. No exam had one.

| # | situation | matches when (for that IST day) | placeholders (*optional*) |
|---|---|---|---|
| 1 | `first_day` | it is the learner's 1st active day | *`questionsToday`* |
| 2 | `camp_completed` | a camp was cleared that day | `campTitle`, `campNumber` |
| 3 | `title_earned` | a title was earned that day (the most significant one is named) | `titleLabel` |
| 4 | `star_earned` | a star was earned that day (the highest level; on a tie, the first subject in rule order) | `subject`, `starLabel` |
| 5 | `accuracy_milestone` | a subject's PYQ accuracy over the last 7 days (≥ 10 answers) went above a rule threshold for the FIRST time. Thresholds are every `subject_accuracy` goal's `minPercent` and every star level's `minAccuracyPercent`, and "first time" is computed from the stored attempt history | `subject`, `percent`, `threshold`, *`weakestSubject`* |
| 6 | `returning_after_break` | ≥ 3 days without study before this day | `daysAway` |
| 7 | `mock_completed` | a mock was submitted that day (the best score is used) | `percent` |
| 8 | `step_completed` | a step was completed that day | `stepNumber`, `campTitle` |
| 9 | `strong_day` | ≥ 50 answers that day | `questionsToday`, `accuracyToday` |
| 10 | `weak_subject` | every 3rd active day, when the weakest subject is below 50%. Weakest = all PYQ attempts up to that day, among subjects with at least the judge's `minAnswers` answers | `weakestSubject`, `weakestPercent` |
| 11 | `default` | always (the fallback; keep templates that use only `{dayNumber}`) | — |
| — | `story_so_far` | the one note for the history before the first open | `activeDays`, `answers`, `topics` |
| — | `stage_cleared` (v7) | written once when the learner taps "Begin Phase 2" — an EXTRA entry that day (the regular note stays) | `stage`, `nextStage`, *`daysOnPath`* (from `stageNotes.minDaysOnPath` up, default 30), *`syllabusPercent`* (from `stageNotes.minSyllabusPercent` up, default 50), *`starsEarned`* (from 1 up) |
| — | `stage_restart` (v7) | written once when the learner restarts Phase 1 after "did not clear" — an extra entry that day | `stage`, *`daysOnPath`*, *`syllabusPercent`* (same floors), *`starsEarned`* |

For the two stage notes the **first** template whose placeholders can all be filled is used (no
rotation), so list them most specific first and end with one that needs no optional number. If
none fits, a `default` template is used. Since rules version 8 the numbers are offered only from
the `stageNotes` floors up, so the choice follows the learner's numbers: the defaults are

| `stage_cleared` (in order) | needs | `stage_restart` (in order) | needs |
|---|---|---|---|
| `.1` "147 days. 89% of the syllabus. Prelims was just the first gate…" | days ≥ 30 and coverage ≥ 50% | `.1` "You showed up for 147 days…" | days ≥ 30 |
| `.4` "62% of the syllabus covered. Prelims was just the first gate…" | coverage ≥ 50% | `.3` "62% of the syllabus is already behind you…" | coverage ≥ 50% |
| `.2` "45 days on the path. Prelims was just the first gate…" | days ≥ 30 | `.4` "One result does not undo the work you put in…" | nothing |
| `.5` "You cleared Prelims. That is the gate that matters. The Summit is a different kind of climb, and we start it together." | nothing | | |

A learner with 9 days and 0% gets the plain one (version 7 rendered "9 days. 0% of the
syllabus."). Version 7's `stage_cleared.3` and `stage_restart.2` were retired; notes already
written keep their text. No stage template names the streak (owner ruling).

The day counts (3, 50, 10 and every 3rd day) and the 60-notes-per-call backfill bound are code
constants in v1.

### Title conditions

`titles[].when` is one of:
- `camp_completed {camp}`: that camp was cleared, in any run.
- `stage_completed {stage}`: every camp of the stage has been cleared, **or** (rules v7) the
  learner moved past it by clearing the real exam ("Begin Phase 2") — whatever share of it was done.
- `active_days {days}`: lifetime distinct active days.
- `stars_total {count}`: lifetime subject stars.
- `streak_days {days}`: still valid, but used by **no** default.

A title is earned once and kept forever, and it survives a restart. Order titles from easiest to
hardest. The **last earned in rule order** is the "most significant" one: it is the one a veteran's
first open celebrates, and it is shown as the current title when several titles arrive together.

---

## 7. Switching the path on for an exam

```http
PATCH /sme/exams/upsc-cse
{ "featureFlags": { "dashboard.activityPath": true } }
```

`true` **removes** the stored `false`, which puts the key back to its registry default (on);
`false` switches it off again. This is the same call and the same rules as every other feature
flag ([SME_EXAMS_API.md](./SME_EXAMS_API.md) §7). The app's `GET /activity-path/path` answers
`403 FEATURE_DISABLED` while it is off. An exam with no rules returns `available: false` even
when switched on.

---

## 8. OPEN — content team (subject mapping)

Owner ruling 2026-11-16 mapped `polity` → "Polity & Governance", `environment` → "Environment",
and `ir_security` → "International Relations" + "Internal Security". **Still open:**

- **Governance & Society ↔ Indian Society?** An unconfirmed guess, so NOT mapped. §5 shows the call if confirmed.
- **Art & Culture, Current Affairs and CSAT** have no study-material subject: questions only.
- **Ethics** (study material) has no PYQ subject and is attached to no path subject.
- **Mock subject "Governance"** (production: 15 questions, 948 submitted answers): Polity (whose
  folder is "Polity & Governance") or Governance & Society? NOT mapped. The other mock spellings
  are mapped (rules version 6) as follows:
  - Polity ← Polity
  - Economy ← Economics, Economy
  - History, Geography and Environment ← their own names
  - Science & Technology ← Science and Technology
  - IR & Security ← International Relations, Internal Security
  - Art & Culture ← Art and Culture

  Current Affairs and CSAT have no mock subject.

---

## 9. The Prelims result (rules version 7)

What the learner sees is described in the app contract
([activity-path-v1.md](../contracts/activity-path-v1.md#prelims-result-and-the-stage-change-b7)).
This section covers what the business sets and what support can do.

### What the business must set

| field | effect | when |
|---|---|---|
| `prelims.resultDate` | From this IST day, the "Did you clear Prelims?" prompt appears by itself for every learner in the first (Prelims) stage who has not answered. It is also the **cycle**: a learner who restarted after this result is not asked again until the date moves. With `null` the prompt never appears by itself, but a learner can still answer from My stats once `exams.exam_date` has passed. A date in the future keeps the question closed | set it on result day; **set the new date every year** |
| `calibrationSimulationId` | the simulation offered as the calibration test after "No". It must exist, be **active** and belong to the exam, otherwise the card reads unavailable and only "Skip for now" works. Its own `totalQuestions` and `durationMinutes` are shown (the path never invents a duration) | before result day |
| `prelims.text.<key>` (optional) | any line of the three screens, en + hi, merged by key. Allowed placeholders per key are in `constants/activity-path-prelims-text.ts`, and the validator rejects anything else | any time |

Example (dry run first):

```http
PATCH /sme/activity-path/rules/upsc-cse
{
  "override": {
    "prelims": { "resultDate": "2027-06-12" },
    "calibrationSimulationId": "ab3550c7-35db-4319-944b-5a2cd39aabce"
  },
  "dryRun": true
}
```

`exams.exam_date` is **not** changed by the path, and it is not a path rule. The "not cleared"
screen mentions it only when it is already in the future ("Your countdown now points to the next
exam on …"). Keep it current as usual (`PATCH /sme/exams/:id`).

### What happens to a learner

- **The answer** is recorded, and nothing else moves. It can be **changed** (the other button)
  until the learner taps "Begin Phase 2" or restarts. After that it is final.
- **"Yes" + Begin**: a `stage` ledger row is written with the Phase 1 numbers frozen (days on the
  path, syllabus %, best streak, stars, stage %). A `stage_cleared` journal note is written, and the
  learner moves to the next stage in the same run. They do not need to have finished every Phase 1
  step. "Completed The Ascent" titles are earned.
- **"No" + restart**: a `stage` row is written with the numbers and the calibration frozen. A
  `stage_restart` note is written. The learner starts run `attempt + 1` of Phase 1 from Base Camp.
  When the calibration test was taken, the "weakest subject" steps are filled from it. Nothing is
  deleted: stamps, stars, titles and journal notes all stay. History counts, so earlier work
  re-clears the early steps. On the "No" screen, "Current camp position" is the number of camps
  reached in this run under the current rules — the same number My stats shows as "Camp N" ("Camp 0
  • On the way to Base Camp" = `prelims.text.keptCampNoneValue`) — and "Stamps" counts every stamp
  ever earned. After a rule edit the two can differ; each keeps its own meaning (rules version 8).
- The streak is never read or reset by any of this.

### Support: undoing an answer

There is no endpoint, because it is rare and needs judgement. Both are **production writes**: run
them by hand, for one learner, after reading the row.

**Before** the learner tapped Begin or restarted (the app also lets them change it themselves):

```sql
UPDATE activity_path_progress
SET prelims_result = NULL, prelims_answered_at = NULL, updated_at = CURRENT_TIMESTAMP
WHERE user_id = '<user id>' AND exam_id = 'upsc-cse';
```

**After "Begin"** (they tapped Yes by mistake and are in The Summit), put them back in Phase 1 with
no answer:

```sql
UPDATE activity_path_progress
SET stage = 'ascent', slot_fills = '{}'::jsonb, prelims_result = NULL, prelims_answered_at = NULL,
    updated_at = CURRENT_TIMESTAMP
WHERE user_id = '<user id>' AND exam_id = 'upsc-cse';
```

The `stage` ledger row stays as history (`/stats` `history` keeps listing it) and so does the title.
If the learner then answers "No" and restarts **in the same run**, the restart's `stage` row would
collide with that kept row and be skipped, so their history would miss the restart. Tell
engineering before doing this.

**After a restart**, the run number has moved on. Setting `attempt` back is not supported: the
ledger is keyed by attempt. Leave the learner on the new run. If they did clear, they can answer
"Yes" with the next result date, or engineering can move them by hand.

Columns involved: `activity_path_progress.stage`, `attempt`, `slot_fills`, `prelims_result`
(CHECK: NULL, `cleared` or `not_cleared`), `prelims_answered_at`, `calibration_attempt_id`, plus
`updated_at`, which must always be set on a raw write. Ledger rows (`activity_path_earned`) are
never updated or deleted.
