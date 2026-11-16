# Activity path — API contract v1

`GET /activity-path/path` — backend item B2 (2026-11-16); subject stars added by B3.
`GET /activity-path/stats` — backend item B3 (2026-11-16), see [/stats](#get-activity-pathstats) below.
`GET /activity-path/journal` and `POST /activity-path/celebrations/seen` — backend item B4, see
[/journal](#get-activity-pathjournal) and [/celebrations/seen](#post-activity-pathcelebrationsseen).
`GET /activity-path/score` — backend item B5, see [/score](#get-activity-pathscore).
B6: the [confidence judge](#confidence-judge-b6) (`confidence` on `/score` subjects + topics and
`/stats` journey rows) and the optional [answer time](#post-pyqidsubmit-and-post-pyqwrong-answersquestionidretry--answer-time-b6)
on PYQ submits (**deploy the backend first**).
B7: the [stage change after Prelims](#prelims-result-and-the-stage-change-b7) — `prelims` on
`/path` and `/stats`, `history` on `/stats`, and `POST /activity-path/prelims-result`,
`POST /activity-path/begin-next-stage`, `POST /activity-path/restart`.
Consumed by the mobile lane. Rules version **10** (B3: star review/mock criteria; B4: titles and
Monk's journal notes; B5: pillar explanations and the topic "Weak" thresholds; B6: judge v1 fields
and a note copy pass; B7: the Prelims-flow copy and the two stage notes; v8: the stage notes fit
the learner's numbers, and "Current camp position" is the same number as the My stats "Camp"
counter — see [Camp numbers](#camp-numbers--one-meaning-per-field)). Examples recorded before
B7 show `"rulesVersion"` 4–6 and have no `prelims` / `history` keys; examples recorded under v7 show
`"rulesVersion": 7`. Apart from the keys each item added, nothing else in their shape changed; the
B7 examples' values that v8 changes were updated by hand. Rules version **9** changes Hindi copy
only (English and every shape unchanged): the server uses the app's terms — फ़ेज़ for a Phase (चरण
stays a Step), डायरी, सवाल, and "Monk" / "Streak" in Latin letters; the Hindi strings in the
examples below were updated by hand, their `"rulesVersion"` values are as recorded. Rules version
**10** (Hindi only again): सफ़र (not यात्रा), "यह पल शेयर करें", "डायरी के नोट", the range label
"इस हफ़्ते"; and a Hindi count phrase with a word that agrees with the number takes the singular at
exactly 1 ("1 सवाल हल किया", "3 में से 1 पूरा", "पिछले 1 दिन में").

- **Auth:** JWT (global guard). **Gate:** `@Feature('dashboard.activityPath')` → `403 FEATURE_DISABLED`
  when the exam has the path switched off (migration `20261116000000` seeded it OFF for every exam).
- **Exam:** `?exam=` / `X-Exam` (default `upsc-cse`). **Language:** `X-Lang` (`en` | `hi`; anything
  else renders English). Response carries `Vary: X-Lang`. Not cached.
- **Side effects:** the call records what the learner has earned (idempotent, append-only) and
  freezes "smart slot" subjects. Calling it twice is safe.
- **Shape rule:** every key below is ALWAYS present. Nullable fields are marked `| null`; lists are
  never null (`[]`). All strings are final (placeholders filled, language chosen) — render as is.

## Root

| field | type | notes |
|---|---|---|
| `available` | boolean | `false` when the exam has no path rules (today: every exam except `upsc-cse`). Then every other field is `null` / `[]` / `false`, except `serverDate`. |
| `rulesVersion` | int \| null | rule-set version |
| `stage` | Stage \| null | the learner's current stage |
| `attempt` | int \| null | 1 = first run; a restart after "did not clear Prelims" increments it (B7) |
| `camps` | Camp[] | camps of the current stage, in order |
| `current` | Current \| null | null when unavailable OR when every step of the stage is done |
| `stageComplete` | boolean | every step of the stage is done. Nothing moves by itself: the next stage begins only through the Prelims answer (`prelims`, B7), and that does **not** need `stageComplete` |
| `chips` | Chips \| null | null only when unavailable |
| `celebrations` | Celebration[] | unseen earned items to celebrate |
| `prelims` | Prelims \| null | B7: the "Did you clear Prelims?" state, see [Prelims result](#prelims-result-and-the-stage-change-b7); null only when unavailable |
| `serverDate` | string | today in IST, `YYYY-MM-DD` |

**Stage** — `key` string, `label` string, `examPart` `"prelims"|"mains"`, `number` int (1-based),
`totalStages` int.

**Camp** — `key` string, `number` int, `label` string, `stampLabel` string (the stamp it awards),
`state` `"cleared"|"current"|"locked"`, `gateText` string \| null, `steps` Step[].

`gateText` is a **derived hint, not a gate** (owner ruling 2026-11-16: there is no camp-gate
rule; a camp opens when the previous camp's steps are done). It is set on **one camp only**:
the first locked camp after the current one. Every other camp, and every camp once the stage is
complete, has null. The text names the hardest unmet goal of the current camp's **last** step.
"Hardest" means the first `subject_accuracy` goal, otherwise the first goal.
- On that last step with the goal in progress: the goal's own progress wording ("Accuracy at 31% • Target 50%").
- Otherwise: the rule's requirement ("50% accuracy in Economy required"). A slot goal that is
  not resolved yet names the slot ("35% accuracy in your weakest subject required").

**Step** — `key` string, `number` int, `state` `"done"|"current"|"locked"`.

**Current** — `campKey`, `campNumber` (the number of the camp the current step belongs to, i.e. the
camp being climbed **to** — mid-stage it is `/stats` `header.campNumber` + 1; see
[Camp numbers](#camp-numbers--one-meaning-per-field)), `stepKey`, `stepNumber`, `stepTitle` (string, e.g. "Step 1"),
`stepsInCamp` int, `tasksLeft` int (goals not done, locked ones included), `goals` Goal[].

**Goal** — `key` string, `type` string (`subject_accuracy`, `topics_completed`, `active_days`,
`mock_score`, `questions_attempted`, `confidence_gap_cleared`, `mains_evaluated`; also
`streak_days`, which is valid but not used by the defaults), `title` string,
`progressText` string, `state` `"done"|"in_progress"|"locked"`, `current` int (rounded down; 0 for
a locked goal), `target` int, `unit` `"percent"|"topics"|"days"|"questions"|"evaluations"|"verdict"`,
`link` string \| null (in-app `/open/…`; **null when the goal is done or locked**).
`locked` = the step's last goal, shown only after the others are done.

**Chips** — `stagePercent` int 0–100 (steps done ÷ steps in stage), `starsEarned` int (subject
stars held, **lifetime**: stars are forever and survive a path restart, so this is NOT per
`attempt`; always equal to `/stats` `stars.earned`), `starsTotal` int (3 × subjects in the rules;
33 for UPSC).

**Celebration** — `id` string (ledger row id; pass it to `POST /activity-path/celebrations/seen`
once shown), `kind` `"camp"|"title"|"star"`, `key` string, `title` string, `body` string \| null,
`number` int \| null. Ordered oldest pass first; within one pass camps (in path order), then
titles, then stars.

| kind | `key` | `title` | `body` | `number` |
|---|---|---|---|---|
| camp | camp key | camp name ("Foothills") | stamp label ("Foothills crossed") | camp number |
| title | title key (`the_disciplined`) | title label ("The Disciplined") — the "New title earned" screen | title description ("Studied on 21 different days") | null |
| star | `<subjectKey>:<levelKey>` (`history:solid`) | "History: Solid" | the level's description | level 1–3 |

A celebration stays in this list until the app marks it seen. On a learner's FIRST open,
everything earned during catch-up (steps, camps, stars, titles, notes) is pre-marked seen EXCEPT
the highest camp and the single **most significant title**, meaning the one listed last in the
rules among those just earned (the defaults go from easiest to hardest). A veteran sees at most two
celebrations, not six camps, twenty stars and four titles. Anything earned on a later open celebrates
normally. The current title is shown on the profile summary (`/stats` `header.title`). `/path` does
not carry it, because the path screen does not show it.

## Semantics worth knowing

- **Sticky:** a goal or step once earned stays done even if the number later drops (e.g. a streak ends).
- **Catch-up:** history counts. One call advances through every step the data already satisfies.
- **The path does not depend on the streak** (owner ruling 2026-11-16, rules version 2). Former
  streak goals are now **active-day goals**: the number of distinct IST days on which the learner
  studied in this exam. A day counts if the learner did any of these in this exam that day:
  answered a PYQ, first answered or last re-answered a library MCQ, opened, last progressed or
  completed a document, submitted a mock, or had a Mains answer evaluated. History counts. A goal
  may be windowed: "Study on 15 of the last 21 days", progress text "13 of 15 days (last 21 days)".
  `streak_days` still exists for rule overrides. When used, it counts the streak only while it is
  alive (credited today or yesterday, IST).
- **Mock score** = best submitted simulation attempt's accuracy (correct ÷ total questions) × 100.
- **Subject stars are forever** (B3). Each subject has 3 levels (Covered, Solid, Mastered), climbed
  in order. A level is recorded once and never taken back, even if accuracy later drops. A path
  restart (`attempt` + 1 after "did not clear Prelims") keeps every star. Stars are evaluated in the
  same call that records goals; the exact rule is in the `/stats` section.
- **Titles are forever** (B4). Like stars, a title is recorded once and never taken back, and it
  survives a restart. The default titles depend on camps, stages, active days and lifetime stars,
  never on the streak.
- **Monk's journal** (B4): one note per **active** IST day, written once and never changed. It is
  a server template filled with that day's real numbers; there is no AI. See
  [/journal](#get-activity-pathjournal).
- **Confidence gap** (`confidence_gap_cleared`) is judged by the **v1 judge with six signals**
  (B6, see [Confidence judge](#confidence-judge-b6)). The goal is met when the subject is
  **cleared**: no gap, and a gap score of 30 or less. With fewer than 20 answers in the subject it
  reads "Answer N more questions to check". With enough answers but no evidence yet (no study
  material, no re-attempt, no timed answer, no mapped mock) it reads "Retry questions you got wrong,
  or take a mock, to check". A goal already earned stays earned, including one earned under the
  old v0 judge.
- **After a rule edit (B6b) — what learners who already earned something see.** A rule edit
  by the business is not a restart. The rule:
  - **Earned rows are never deleted or rewritten.** That covers goals, steps, camp stamps,
    stars, titles and journal notes. `/stats` `stamps[]` keeps every stamp, the current title
    stays, a stamped camp still counts for `camp_completed` / `stage_completed` titles, and
    notes are history. All of these were awarded legitimately under the rules of the day.
  - **The path follows the CURRENT rules.** A camp is `cleared` only while every one of its
    current steps is done. A camp the learner stamped earlier, but which the edit made longer or
    moved ahead of the current step, reads `current` / `locked`. `/stats` `header.campNumber`
    and the not-cleared screen's "Current camp position" count cleared camps the same way, while
    "Stamps" keeps counting the stamp (rules v8, [Camp numbers](#camp-numbers--one-meaning-per-field)).
  - **Celebrations never contradict the path.** `/path` returns a celebration only if the current
    rules agree with it:
    - camp: the camp is in the current stage and all its current steps are done;
    - star: its subject and level still exist;
    - title: it still exists, and a camp/stage title needs that camp/stage cleared when it is in
      the current stage;
    - every kind: the stored text is non-empty.
  - **A withheld celebration stays pending.** It is not marked seen. When the learner clears
    that camp again under the rules in force, the same ledger row plays then; nothing is written
    twice. Proven on real Postgres: an override shrank Base Camp, the learner cleared it, then
    the override was removed. `/path` showed Base Camp current with no celebration. With the
    override back, the camp and "The Trailblazer" played. With the title removed from the rules,
    only the camp played.
- **Smart slots** ("your weakest subject") are chosen once, when the step becomes current, and never move.
- `{roadmapSubject}` links open the FIRST study-material folder of a multi-folder subject (IR & Security → "International Relations").
- **Mock links:** mock-test goals link to `/open/simulation_start` (no id: the Simulation list);
  the B7 calibration card links to `/open/simulation_start/<simulationId>` (starts that test). The
  mobile tree routes both (`DeepLinkTarget.TYPE_SIMULATION_START`); an app build without it opens Home.

### Camp numbers — one meaning per field

Rules v8 (device test 2026-11-16: the not-cleared screen said "Camp 1 • Base Camp" while My stats
said "Camp 0" for the same learner). There are three different things, and each field states
exactly one. **Position** ("Camp N") = camps of the current stage **cleared in the current run
under the current rules**; 0 = still climbing to the first camp. Cleared camps are always the first
N camps of the stage, so N is also the number of the camp last reached. **Identity** = which camp
(1 = Base Camp). **Stamps** = stamps ever earned: kept forever, across runs and rule edits, so they
can exceed the position (a stamp from an earlier rule set, or from an earlier run).

| field | meaning |
|---|---|
| `/stats` `header.campNumber` | **position**: camps reached in this run ("Camp 0", "Camp 2") |
| `/prelims-result` `notCleared.kept[camp]` (`value`, `text`) | **position**, the same number: "Camp 2 • Foothills" (the camp last reached), or "Camp 0 • On the way to Base Camp" when none is reached yet |
| `/prelims-result` `notCleared.kept[stamps]` | **stamps** ever earned, all runs and stages = `/stats` `stamps.length` |
| `/stats` `stamps[]` | every stamp ever earned; each `number` is that camp's **identity** when stamped |
| `phaseNumbers.campsCleared` (cleared screen, `stage` ledger row, `/stats` `history[].numbers`) | **position** frozen when the stage ended = `header.campNumber` at that moment |
| `phaseNumbers.stampsEarned` | stamps of **that stage earned in that run** (may exceed `campsCleared` after a rule edit) |
| `/path` `camps[].number`, `celebrations[camp].number` | **identity** |
| `/path` `current.campNumber` | **identity** of the camp the current step belongs to (being climbed **to**) = position + 1 mid-stage |
| journal `camp_completed` `{campNumber}` / `{campTitle}`, `step_completed` `{campTitle}` | **identity** of the camp stamped / of the step's camp, frozen in the note |
| titles with `camp_completed` / `stage_completed` conditions | **stamps** ever earned (any run) |

## Filled later (shape already final)

Nothing left: B7 filled the stage change, the `stageComplete` follow-up and `attempt` > 1.

## Example 1 — brand-new learner (real response, throwaway Postgres, en)

The body is complete: all six camps list their five steps. Nothing is earned yet, so `celebrations`
is empty and `stagePercent` is 0.

<!-- example: path-response | rules v7 | brand-new learner, first open, en -->
```json
{
  "available": true,
  "rulesVersion": 7,
  "stage": {
    "key": "ascent",
    "label": "The Ascent",
    "examPart": "prelims",
    "number": 1,
    "totalStages": 2
  },
  "attempt": 1,
  "camps": [
    {
      "key": "ascent.c1",
      "number": 1,
      "label": "Base Camp",
      "stampLabel": "Base Camp reached",
      "state": "current",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c1.s1",
          "number": 1,
          "state": "current"
        },
        {
          "key": "ascent.c1.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c1.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c1.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c1.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    },
    {
      "key": "ascent.c2",
      "number": 2,
      "label": "Foothills",
      "stampLabel": "Foothills crossed",
      "state": "locked",
      "gateText": "35% accuracy in your weakest subject required",
      "steps": [
        {
          "key": "ascent.c2.s1",
          "number": 1,
          "state": "locked"
        },
        {
          "key": "ascent.c2.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c2.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c2.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c2.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    },
    {
      "key": "ascent.c3",
      "number": 3,
      "label": "Ridge Trail",
      "stampLabel": "Ridge Trail conquered",
      "state": "locked",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c3.s1",
          "number": 1,
          "state": "locked"
        },
        {
          "key": "ascent.c3.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c3.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c3.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c3.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    },
    {
      "key": "ascent.c4",
      "number": 4,
      "label": "High Camp",
      "stampLabel": "High Camp reached",
      "state": "locked",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c4.s1",
          "number": 1,
          "state": "locked"
        },
        {
          "key": "ascent.c4.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c4.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c4.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c4.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    },
    {
      "key": "ascent.c5",
      "number": 5,
      "label": "Icefall",
      "stampLabel": "Icefall crossed",
      "state": "locked",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c5.s1",
          "number": 1,
          "state": "locked"
        },
        {
          "key": "ascent.c5.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c5.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c5.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c5.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    },
    {
      "key": "ascent.c6",
      "number": 6,
      "label": "Final Ridge",
      "stampLabel": "Prelims summit reached",
      "state": "locked",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c6.s1",
          "number": 1,
          "state": "locked"
        },
        {
          "key": "ascent.c6.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c6.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c6.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c6.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    }
  ],
  "current": {
    "campKey": "ascent.c1",
    "campNumber": 1,
    "stepKey": "ascent.c1.s1",
    "stepNumber": 1,
    "stepTitle": "Step 1",
    "stepsInCamp": 5,
    "tasksLeft": 3,
    "goals": [
      {
        "key": "ascent.c1.s1.attempt",
        "type": "questions_attempted",
        "title": "Attempt 10 questions",
        "progressText": "0 of 10 done",
        "state": "in_progress",
        "current": 0,
        "target": 10,
        "unit": "questions",
        "link": "/open/pyq"
      },
      {
        "key": "ascent.c1.s1.topics-least_covered_subject",
        "type": "topics_completed",
        "title": "Complete 1 topic in Polity",
        "progressText": "0 of 1 done",
        "state": "in_progress",
        "current": 0,
        "target": 1,
        "unit": "topics",
        "link": "/open/doc_list?subject=Polity%20%26%20Governance"
      },
      {
        "key": "ascent.c1.s1.active",
        "type": "active_days",
        "title": "Study on 2 different days",
        "progressText": "0 of 2 days",
        "state": "in_progress",
        "current": 0,
        "target": 2,
        "unit": "days",
        "link": "/open/pyq"
      }
    ]
  },
  "stageComplete": false,
  "chips": {
    "stagePercent": 0,
    "starsEarned": 0,
    "starsTotal": 33
  },
  "celebrations": [],
  "prelims": {
    "promptDue": false,
    "canAnswer": false,
    "result": null,
    "answeredDate": null,
    "resultDate": null,
    "changeable": false,
    "nextAction": null,
    "prompt": {
      "title": "Did you clear Prelims?",
      "body": "The result is out. Let me know so I can set up the next phase of your Journey. Either way, your progress stays exactly where it is. Your Journey continues.",
      "yesLabel": "Yes, I cleared Prelims",
      "noLabel": "No, I didn't clear this time"
    }
  },
  "serverDate": "2026-11-16"
}
```

## Example 2 — veteran, FIRST open (real response, en, rules v7)

The seed: 275 PYQ answers over 8 active IST days and 8 completed documents. This is the learner's first call ever. It cleared Camp 1 (Base Camp) and left Camp 2 Step 1 current, with two of its three goals done (`tasksLeft` 1). It recorded 10 stars, two titles and two journal notes (a "story so far" for the 7 earlier days, then today's). Only the highest camp and the most significant title stay unseen: Base Camp and The Star Collector. The Trailblazer and all 10 stars were pre-marked seen. The body is complete: every camp lists its five steps.

<!-- example: path-response | rules v7 | veteran (275 PYQ answers over 8 days, 8 documents), FIRST open, en -->
```json
{
  "available": true,
  "rulesVersion": 7,
  "stage": {
    "key": "ascent",
    "label": "The Ascent",
    "examPart": "prelims",
    "number": 1,
    "totalStages": 2
  },
  "attempt": 1,
  "camps": [
    {
      "key": "ascent.c1",
      "number": 1,
      "label": "Base Camp",
      "stampLabel": "Base Camp reached",
      "state": "cleared",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c1.s1",
          "number": 1,
          "state": "done"
        },
        {
          "key": "ascent.c1.s2",
          "number": 2,
          "state": "done"
        },
        {
          "key": "ascent.c1.s3",
          "number": 3,
          "state": "done"
        },
        {
          "key": "ascent.c1.s4",
          "number": 4,
          "state": "done"
        },
        {
          "key": "ascent.c1.s5",
          "number": 5,
          "state": "done"
        }
      ]
    },
    {
      "key": "ascent.c2",
      "number": 2,
      "label": "Foothills",
      "stampLabel": "Foothills crossed",
      "state": "current",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c2.s1",
          "number": 1,
          "state": "current"
        },
        {
          "key": "ascent.c2.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c2.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c2.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c2.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    },
    {
      "key": "ascent.c3",
      "number": 3,
      "label": "Ridge Trail",
      "stampLabel": "Ridge Trail conquered",
      "state": "locked",
      "gateText": "45% accuracy in Polity required",
      "steps": [
        {
          "key": "ascent.c3.s1",
          "number": 1,
          "state": "locked"
        },
        {
          "key": "ascent.c3.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c3.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c3.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c3.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    },
    {
      "key": "ascent.c4",
      "number": 4,
      "label": "High Camp",
      "stampLabel": "High Camp reached",
      "state": "locked",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c4.s1",
          "number": 1,
          "state": "locked"
        },
        {
          "key": "ascent.c4.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c4.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c4.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c4.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    },
    {
      "key": "ascent.c5",
      "number": 5,
      "label": "Icefall",
      "stampLabel": "Icefall crossed",
      "state": "locked",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c5.s1",
          "number": 1,
          "state": "locked"
        },
        {
          "key": "ascent.c5.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c5.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c5.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c5.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    },
    {
      "key": "ascent.c6",
      "number": 6,
      "label": "Final Ridge",
      "stampLabel": "Prelims summit reached",
      "state": "locked",
      "gateText": null,
      "steps": [
        {
          "key": "ascent.c6.s1",
          "number": 1,
          "state": "locked"
        },
        {
          "key": "ascent.c6.s2",
          "number": 2,
          "state": "locked"
        },
        {
          "key": "ascent.c6.s3",
          "number": 3,
          "state": "locked"
        },
        {
          "key": "ascent.c6.s4",
          "number": 4,
          "state": "locked"
        },
        {
          "key": "ascent.c6.s5",
          "number": 5,
          "state": "locked"
        }
      ]
    }
  ],
  "current": {
    "campKey": "ascent.c2",
    "campNumber": 2,
    "stepKey": "ascent.c2.s1",
    "stepNumber": 1,
    "stepTitle": "Step 1",
    "stepsInCamp": 5,
    "tasksLeft": 1,
    "goals": [
      {
        "key": "ascent.c2.s1.attempt",
        "type": "questions_attempted",
        "title": "Attempt 150 questions",
        "progressText": "Completed",
        "state": "done",
        "current": 275,
        "target": 150,
        "unit": "questions",
        "link": null
      },
      {
        "key": "ascent.c2.s1.accuracy-environment",
        "type": "subject_accuracy",
        "title": "Achieve 40% PYQ accuracy in Environment & Ecology (20+ questions)",
        "progressText": "Completed",
        "state": "done",
        "current": 60,
        "target": 40,
        "unit": "percent",
        "link": null
      },
      {
        "key": "ascent.c2.s1.topics-geography",
        "type": "topics_completed",
        "title": "Complete 2 topics in Geography",
        "progressText": "1 of 2 done",
        "state": "in_progress",
        "current": 1,
        "target": 2,
        "unit": "topics",
        "link": "/open/doc_list?subject=Geography"
      }
    ]
  },
  "stageComplete": false,
  "chips": {
    "stagePercent": 16,
    "starsEarned": 10,
    "starsTotal": 33
  },
  "celebrations": [
    {
      "id": "c566f30e-8011-49eb-8d07-7f87180af173",
      "kind": "camp",
      "key": "ascent.c1",
      "title": "Base Camp",
      "body": "Base Camp reached",
      "number": 1
    },
    {
      "id": "aee25d0d-848d-478e-979c-fe15c8e2a9cb",
      "kind": "title",
      "key": "the_star_collector",
      "title": "The Star Collector",
      "body": "Earned 10 subject stars",
      "number": null
    }
  ],
  "prelims": {
    "promptDue": false,
    "canAnswer": false,
    "result": null,
    "answeredDate": null,
    "resultDate": null,
    "changeable": false,
    "nextAction": null,
    "prompt": {
      "title": "Did you clear Prelims?",
      "body": "The result is out. Let me know so I can set up the next phase of your Journey. Either way, your progress stays exactly where it is. Your Journey continues.",
      "yesLabel": "Yes, I cleared Prelims",
      "noLabel": "No, I didn't clear this time"
    }
  },
  "serverDate": "2026-11-16"
}
```

## Example 3 — exam without path rules

<!-- example: path-response | rules v7 | exam without path rules (appsc-group-1) -->
```json
{
  "available": false,
  "rulesVersion": null,
  "stage": null,
  "attempt": null,
  "camps": [],
  "current": null,
  "stageComplete": false,
  "chips": null,
  "celebrations": [],
  "prelims": null,
  "serverDate": "2026-11-16"
}
```

---

## GET /activity-path/stats

The "My stats" and profile-summary screens, plus the "Your subject stars" sheet. Backend item B3.

- **Auth / gate / exam / language / caching:** exactly as `/path` (JWT, `@Feature('dashboard.activityPath')`
  → `403 FEATURE_DISABLED`, `?exam=` / `X-Exam`, `X-Lang`, `Vary: X-Lang`, not cached).
- **Query:** `range` = `week` (default) | `month` | `all`. Anything else (including `WEEK` or a
  repeated `range`) → `400 {"success": false, "message": "range must be one of: week, month, all"}`.
- **Side effects — the same as `/path`.** `/stats` runs the **same locked catch-up pass** as `/path`
  (goals, steps, camps, stars, slot fills, the first-open rule), then renders a different screen. So
  every star or stamp it shows is recorded, and `/stats` then `/path` leaves exactly the ledger
  `/path` alone would. This was proven on real Postgres, including the two calls running at once.
  `/stats` never returns celebrations: they stay unseen for `/path`.
- **Shape rule:** as `/path`. Every key is always present, lists are never null, and every string is final.

### Root

| field | type | notes |
|---|---|---|
| `available` | boolean | `false` when the exam has no path rules: then `header`, `stars`, `performance` are `null`, `stamps`/`journey` are `[]` |
| `rulesVersion` | int \| null | |
| `header` | Header \| null | profile summary |
| `stamps` | Stamp[] | every camp stamp earned, **across runs** (kept after a restart), oldest run first, in camp order |
| `stars` | Stars \| null | the subject-stars sheet |
| `journey` | JourneyItem[] | one per subject, in rule order (11 for UPSC) |
| `performance` | Performance \| null | "Your performance" for `range` |
| `journal` | {`latest`: JournalEntry \| null, `total`: int} \| null | the newest Monk note (the "My stats" card with "View full journal") and how many notes exist; `latest` is null before the first note. JournalEntry is the `/journal` entry shape. B4 |
| `prelims` | Prelims \| null | B7: same object as `/path` `prelims` — My stats is where the learner (re)opens the question (`canAnswer`) |
| `history` | HistoryEntry[] | B7: every completed stage and restarted run, oldest first, with its numbers frozen when it happened ("phase 1 scores are saved"). `[]` until the first one. Optional for the app to render |
| `serverDate` | string | today in IST, `YYYY-MM-DD` |

**Header**

| field | type | meaning |
|---|---|---|
| `title` | string \| null | the learner's current title, meaning the most recently earned one ("The Disciplined"); titles earned in the same call → the last in rule order. Rendered from the stored snapshot, so a rule edit never renames a title already held. null when none (B4) |
| `titleKey` | string \| null | its key (`the_disciplined`); null when none |
| `dayNumber` | int ≥ 1 | the IST study day the learner is on = past active days + today. Brand-new learner → 1; first activity today → 1; 5 earlier active days and nothing yet today → 6 |
| `dayLabel` | string | "Day 18 • 16 Nov" / "दिन 18 • 16 नवंबर" (today, IST) |
| `startedDate` | string | first active IST day (history counts, so it can predate the path); with no activity yet, the IST day the path was first opened |
| `stageLabel` | string | current stage ("The Ascent") |
| `nextStageLabel` | string \| null | next stage ("The Summit"); null on the last stage |
| `stagePercent` | int 0–100 | = `/path` `chips.stagePercent` |
| `campNumber` | int | the design's "Camp" counter = camps **reached** (cleared) in the current stage, current run, current rules; 0 = still climbing to the first camp. Same number as the not-cleared screen's "Current camp position"; not the stamp count ([Camp numbers](#camp-numbers--one-meaning-per-field)) |
| `stepsLeft` | int | steps left to clear the current camp (current step included); 0 when the stage is complete |
| `streak` | int | **display only**: the stored streak while alive (credited today or yesterday, IST), else 0. Nothing on the path, the stars or the stats reads it |

**Stamp** — `campKey` string, `number` int \| null (camp number when earned), `label` string (camp
name), `stampLabel` string \| null, `earnedDate` string (IST day), `attempt` int (the run, 1 = first).
Rendered from the ledger snapshot, so a later rule edit never rewrites a stamp.

**Stars** — `earned` int (= `/path` `chips.starsEarned`), `total` int (3 × subjects), `levels`
StarLevel[3], `subjects` SubjectStars[] (rule order).
**StarLevel** — `key` `"covered"|"solid"|"mastered"`, `level` int 1–3, `label` string,
`description` string (the explanatory card, placeholders filled from the level's numbers).
**SubjectStars** — `key`, `label`, `stars` int 0–3.

**JourneyItem**

| field | type | meaning |
|---|---|---|
| `key`, `label` | string | the path subject |
| `topicsDone` | int | completed study documents in the subject's folder(s), in the language the Library roadmap shows; 0 for an unmapped subject |
| `topicsTotal` | int \| null | **null when the subject has no study material mapped** (Art & Culture, Governance & Society, Current Affairs, CSAT); 0 if mapped but the folder has no documents |
| `percent` | int \| null | the roadmap's completion % (partial progress counts, rounded down), the same number the Library shows; null when unmapped or no documents |
| `status` | `"not_started"\|"in_progress"\|"completed"` | see below |
| `statusLabel` | string | "Not started" / "In progress" / "Completed" (hi: "शुरू नहीं हुआ" / "जारी है" / "पूरा हुआ") |
| `progressText` | string | "3 of 12 topics"; for an unmapped subject "45 questions answered" |
| `stars` | int 0–3 | as in `stars.subjects` |
| `link` | string \| null | `/open/doc_list?subject=<first folder>` when mapped, else `/open/practice_list?filterType=subject&filterValue=<PYQ subject>&examType=prelims` (same encoding as goal links) |
| `confidence` | Confidence | **B6.** The subject-level judge verdict, the same object as `/score` `subjects[].confidence` (see [Confidence judge](#confidence-judge-b6)). Older examples in this doc predate it |

`status`: a subject **with documents** is `completed` when every document is completed, and
`in_progress` once any topic, partial reading or question exists. A subject **without** documents
(unmapped, or a mapped folder with no documents) is judged on question activity: `in_progress`
once a question is answered, and `completed` only once it holds all 3 stars.

**Performance**

| field | type | meaning |
|---|---|---|
| `range` | `"week"\|"month"\|"all"` | echoed |
| `label` | string | "This week" / "This month" / "All time" (hi: "इस हफ़्ते" / "इस महीने" / "अब तक") |
| `fromDate` | string \| null | first IST day counted: week = last **7** IST days including today, month = last **30**; null for all |
| `accuracyPercent` | int \| null | correct ÷ answered over the questions below, rounded down; **null when nothing was answered in range** |
| `questionsAttempted` | int | distinct questions (PYQ + library MCQ, this exam, playable/published only) whose **latest answer** falls in the range, scored by that answer. Because every range ends today, for PYQs this is the same as "answered at least once in range". Library MCQs keep one row per question, so a library question counts if it was **first or last** answered in range; re-answers in between are not stored. With `all` it equals the `/path` "Attempt N questions" count |
| `topicsCompleted` | int | documents completed in range (`completed_at`; `updated_at` for a completed row without one), in the roadmap's language per folder, **including folders not attached to a path subject** (e.g. Ethics), so it can exceed the sum of `journey[].topicsDone` |

### Subject stars — the rule (rules v3)

Stars are evaluated in the same pass that records goals, from the same snapshot. A subject's level N
can be newly earned only when levels 1..N−1 are held, so a learner holds 1..N stars. Once recorded,
a star is never taken back. A level is met when **every criterion that applies** holds:

| criterion (rule field) | applies when | condition |
|---|---|---|
| coverage (`minCoveragePercent` > 0) | the subject has study material AND documents in this exam | roadmap completion % ≥ minimum |
| accuracy (`minAccuracyPercent` / `minAnswers` > 0) | the subject has a PYQ spelling | answers (PYQ + library, latest per question) ≥ max(1, `minAnswers`) AND accuracy ≥ minimum |
| no confidence gap (`requireNoConfidenceGap`) | the subject has a PYQ spelling | the judge says "clear" (`insufficient_data` does not count) |
| spaced review (`minReviewDays` > 0) | always | distinct IST days the learner re-answered a question of the subject first answered on an EARLIER IST day ≥ minimum |
| mock (`minMockPercent` set, > 0) | the exam has an active mock | best submitted mock accuracy in the exam (not per subject) ≥ minimum |
| some activity | always | at least one answer in the subject, or some roadmap progress (a star is never awarded on no data) |

Defaults (UPSC; the review and mock numbers were chosen by the backend lane, **owner to confirm**):

| level | coverage | accuracy / answers | gap | review days | mock |
|---|---|---|---|---|---|
| 1 Covered | 30% | — / 10 | no | 0 | — |
| 2 Solid | 60% | 50% / 40 | no | 1 | — |
| 3 Mastered | 90% | 65% / 100 | must be clear | 3 | 50% |

A subject without study material (CSAT, Art & Culture, Governance & Society, Current Affairs)
skips the coverage row. Review days count PYQ re-attempts, including replay-feed retries, which
cannot be told apart from ordinary re-attempts. They also count a library MCQ re-answered on a
later day, but only its latest re-answer survives in the data. Mock questions use a third subject
vocabulary with no mapping, so the mock threshold is exam-wide.

## /stats Example A — brand-new learner on Day 1 (real response, throwaway Postgres, en, first call ever)

First call ever, with no activity yet: nothing answered, so `performance.accuracyPercent` is null and
every journey row is `not_started`. No title, no stars and no stamps yet. No journal note has been
written (`journal.total` 0, `latest` null).

<!-- example: stats-response | rules v7 | brand-new learner, en, range=week -->
```json
{
  "available": true,
  "rulesVersion": 7,
  "header": {
    "title": null,
    "titleKey": null,
    "dayNumber": 1,
    "dayLabel": "Day 1 • 16 Nov",
    "startedDate": "2026-11-16",
    "stageLabel": "The Ascent",
    "nextStageLabel": "The Summit",
    "stagePercent": 0,
    "campNumber": 0,
    "stepsLeft": 5,
    "streak": 0
  },
  "stamps": [],
  "stars": {
    "earned": 0,
    "total": 33,
    "levels": [
      {
        "key": "covered",
        "level": 1,
        "label": "Covered",
        "description": "You've read the basics and attempted the first set of questions for this subject"
      },
      {
        "key": "solid",
        "level": 2,
        "label": "Solid",
        "description": "Accuracy above 50% in PYQs and at least one spaced review completed"
      },
      {
        "key": "mastered",
        "level": 3,
        "label": "Mastered",
        "description": "High accuracy, consistent revision, and performance above threshold in mock tests"
      }
    ],
    "subjects": [
      {
        "key": "polity",
        "label": "Polity",
        "stars": 0
      },
      {
        "key": "economy",
        "label": "Economy",
        "stars": 0
      },
      {
        "key": "history",
        "label": "History",
        "stars": 0
      },
      {
        "key": "geography",
        "label": "Geography",
        "stars": 0
      },
      {
        "key": "environment",
        "label": "Environment & Ecology",
        "stars": 0
      },
      {
        "key": "science_tech",
        "label": "Science & Technology",
        "stars": 0
      },
      {
        "key": "ir_security",
        "label": "IR & Security",
        "stars": 0
      },
      {
        "key": "art_culture",
        "label": "Art & Culture",
        "stars": 0
      },
      {
        "key": "governance_society",
        "label": "Governance & Society",
        "stars": 0
      },
      {
        "key": "current_affairs",
        "label": "Current Affairs",
        "stars": 0
      },
      {
        "key": "csat",
        "label": "CSAT",
        "stars": 0
      }
    ]
  },
  "journey": [
    {
      "key": "polity",
      "label": "Polity",
      "topicsDone": 0,
      "topicsTotal": 3,
      "percent": 0,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 of 3 topics",
      "stars": 0,
      "link": "/open/doc_list?subject=Polity%20%26%20Governance",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "economy",
      "label": "Economy",
      "topicsDone": 0,
      "topicsTotal": 5,
      "percent": 0,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 of 5 topics",
      "stars": 0,
      "link": "/open/doc_list?subject=Economy",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "history",
      "label": "History",
      "topicsDone": 0,
      "topicsTotal": 1,
      "percent": 0,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 of 1 topic",
      "stars": 0,
      "link": "/open/doc_list?subject=History",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "geography",
      "label": "Geography",
      "topicsDone": 0,
      "topicsTotal": 1,
      "percent": 0,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 of 1 topic",
      "stars": 0,
      "link": "/open/doc_list?subject=Geography",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "environment",
      "label": "Environment & Ecology",
      "topicsDone": 0,
      "topicsTotal": 1,
      "percent": 0,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 of 1 topic",
      "stars": 0,
      "link": "/open/doc_list?subject=Environment",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "science_tech",
      "label": "Science & Technology",
      "topicsDone": 0,
      "topicsTotal": 1,
      "percent": 0,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 of 1 topic",
      "stars": 0,
      "link": "/open/doc_list?subject=Science%20%26%20Technology",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "ir_security",
      "label": "IR & Security",
      "topicsDone": 0,
      "topicsTotal": 2,
      "percent": 0,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 of 2 topics",
      "stars": 0,
      "link": "/open/doc_list?subject=International%20Relations",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "art_culture",
      "label": "Art & Culture",
      "topicsDone": 0,
      "topicsTotal": null,
      "percent": null,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 questions answered",
      "stars": 0,
      "link": "/open/practice_list?filterType=subject&filterValue=Art%20%26%20Culture&examType=prelims",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "governance_society",
      "label": "Governance & Society",
      "topicsDone": 0,
      "topicsTotal": null,
      "percent": null,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 questions answered",
      "stars": 0,
      "link": "/open/practice_list?filterType=subject&filterValue=Governance%20%26%20Society&examType=prelims",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "current_affairs",
      "label": "Current Affairs",
      "topicsDone": 0,
      "topicsTotal": null,
      "percent": null,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 questions answered",
      "stars": 0,
      "link": "/open/practice_list?filterType=subject&filterValue=Current%20Affairs&examType=prelims",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "csat",
      "label": "CSAT",
      "topicsDone": 0,
      "topicsTotal": null,
      "percent": null,
      "status": "not_started",
      "statusLabel": "Not started",
      "progressText": "0 questions answered",
      "stars": 0,
      "link": "/open/practice_list?filterType=subject&filterValue=CSAT&examType=prelims",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    }
  ],
  "performance": {
    "range": "week",
    "label": "This week",
    "fromDate": "2026-11-10",
    "accuracyPercent": null,
    "questionsAttempted": 0,
    "topicsCompleted": 0
  },
  "journal": {
    "latest": null,
    "total": 0
  },
  "prelims": {
    "promptDue": false,
    "canAnswer": false,
    "result": null,
    "answeredDate": null,
    "resultDate": null,
    "changeable": false,
    "nextAction": null,
    "prompt": {
      "title": "Did you clear Prelims?",
      "body": "The result is out. Let me know so I can set up the next phase of your Journey. Either way, your progress stays exactly where it is. Your Journey continues.",
      "yesLabel": "Yes, I cleared Prelims",
      "noLabel": "No, I didn't clear this time"
    }
  },
  "history": [],
  "serverDate": "2026-11-16"
}
```

## /stats Example B — learner with stars, stamps, a title and a journal (real response, en, `range=week`)

The seed is the same as `/path` Example 2 (275 PYQ answers over 8 active days, 8 completed
documents), read right after that learner's first open. The first open recorded one stamp (Base
Camp) and 10 stars: every subject holds one star (Covered) except Economy, which holds none yet
although its judge verdict is `clear`. The Trailblazer and The Star Collector were earned in the
same call, and `title` shows the later one in rule order. The journal holds two notes (a "story so
far" and today's), and `latest` is today's.

In `performance`, 242 questions fall in the week at 59% accuracy, and 8 topics were completed in
it. `dayNumber` is 8 and `startedDate` is 2026-11-09. `streak` is 0: this seeded learner has no
streak record.

<!-- example: stats-response | rules v7 | the veteran after its first open: stars, a stamp, a title, journal; en, range=week -->
```json
{
  "available": true,
  "rulesVersion": 7,
  "header": {
    "title": "The Star Collector",
    "titleKey": "the_star_collector",
    "dayNumber": 8,
    "dayLabel": "Day 8 • 16 Nov",
    "startedDate": "2026-11-09",
    "stageLabel": "The Ascent",
    "nextStageLabel": "The Summit",
    "stagePercent": 16,
    "campNumber": 1,
    "stepsLeft": 5,
    "streak": 0
  },
  "stamps": [
    {
      "campKey": "ascent.c1",
      "number": 1,
      "label": "Base Camp",
      "stampLabel": "Base Camp reached",
      "earnedDate": "2026-11-16",
      "attempt": 1
    }
  ],
  "stars": {
    "earned": 10,
    "total": 33,
    "levels": [
      {
        "key": "covered",
        "level": 1,
        "label": "Covered",
        "description": "You've read the basics and attempted the first set of questions for this subject"
      },
      {
        "key": "solid",
        "level": 2,
        "label": "Solid",
        "description": "Accuracy above 50% in PYQs and at least one spaced review completed"
      },
      {
        "key": "mastered",
        "level": 3,
        "label": "Mastered",
        "description": "High accuracy, consistent revision, and performance above threshold in mock tests"
      }
    ],
    "subjects": [
      {
        "key": "polity",
        "label": "Polity",
        "stars": 1
      },
      {
        "key": "economy",
        "label": "Economy",
        "stars": 0
      },
      {
        "key": "history",
        "label": "History",
        "stars": 1
      },
      {
        "key": "geography",
        "label": "Geography",
        "stars": 1
      },
      {
        "key": "environment",
        "label": "Environment & Ecology",
        "stars": 1
      },
      {
        "key": "science_tech",
        "label": "Science & Technology",
        "stars": 1
      },
      {
        "key": "ir_security",
        "label": "IR & Security",
        "stars": 1
      },
      {
        "key": "art_culture",
        "label": "Art & Culture",
        "stars": 1
      },
      {
        "key": "governance_society",
        "label": "Governance & Society",
        "stars": 1
      },
      {
        "key": "current_affairs",
        "label": "Current Affairs",
        "stars": 1
      },
      {
        "key": "csat",
        "label": "CSAT",
        "stars": 1
      }
    ]
  },
  "journey": [
    {
      "key": "polity",
      "label": "Polity",
      "topicsDone": 1,
      "topicsTotal": 3,
      "percent": 33,
      "status": "in_progress",
      "statusLabel": "In progress",
      "progressText": "1 of 3 topics",
      "stars": 1,
      "link": "/open/doc_list?subject=Polity%20%26%20Governance",
      "confidence": {
        "state": "clear",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "economy",
      "label": "Economy",
      "topicsDone": 1,
      "topicsTotal": 5,
      "percent": 20,
      "status": "in_progress",
      "statusLabel": "In progress",
      "progressText": "1 of 5 topics",
      "stars": 0,
      "link": "/open/doc_list?subject=Economy",
      "confidence": {
        "state": "clear",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "history",
      "label": "History",
      "topicsDone": 1,
      "topicsTotal": 1,
      "percent": 100,
      "status": "completed",
      "statusLabel": "Completed",
      "progressText": "1 of 1 topic",
      "stars": 1,
      "link": "/open/doc_list?subject=History",
      "confidence": {
        "state": "gap",
        "message": "Confidence is way higher than accuracy.",
        "actionLabel": "Take reality check quiz",
        "link": "/open/replay_session?subject=History"
      }
    },
    {
      "key": "geography",
      "label": "Geography",
      "topicsDone": 1,
      "topicsTotal": 1,
      "percent": 100,
      "status": "completed",
      "statusLabel": "Completed",
      "progressText": "1 of 1 topic",
      "stars": 1,
      "link": "/open/doc_list?subject=Geography",
      "confidence": {
        "state": "gap",
        "message": "Confidence is way higher than accuracy.",
        "actionLabel": "Take reality check quiz",
        "link": "/open/replay_session?subject=Geography"
      }
    },
    {
      "key": "environment",
      "label": "Environment & Ecology",
      "topicsDone": 1,
      "topicsTotal": 1,
      "percent": 100,
      "status": "completed",
      "statusLabel": "Completed",
      "progressText": "1 of 1 topic",
      "stars": 1,
      "link": "/open/doc_list?subject=Environment",
      "confidence": {
        "state": "gap",
        "message": "Confidence is way higher than accuracy.",
        "actionLabel": "Take reality check quiz",
        "link": "/open/replay_session?subject=Environment%20%26%20Ecology"
      }
    },
    {
      "key": "science_tech",
      "label": "Science & Technology",
      "topicsDone": 1,
      "topicsTotal": 1,
      "percent": 100,
      "status": "completed",
      "statusLabel": "Completed",
      "progressText": "1 of 1 topic",
      "stars": 1,
      "link": "/open/doc_list?subject=Science%20%26%20Technology",
      "confidence": {
        "state": "gap",
        "message": "Confidence is way higher than accuracy.",
        "actionLabel": "Take reality check quiz",
        "link": "/open/replay_session?subject=Science%20%26%20Technology"
      }
    },
    {
      "key": "ir_security",
      "label": "IR & Security",
      "topicsDone": 2,
      "topicsTotal": 2,
      "percent": 100,
      "status": "completed",
      "statusLabel": "Completed",
      "progressText": "2 of 2 topics",
      "stars": 1,
      "link": "/open/doc_list?subject=International%20Relations",
      "confidence": {
        "state": "gap",
        "message": "Confidence is way higher than accuracy.",
        "actionLabel": "Take reality check quiz",
        "link": "/open/replay_session?subject=IR%20%26%20Security"
      }
    },
    {
      "key": "art_culture",
      "label": "Art & Culture",
      "topicsDone": 0,
      "topicsTotal": null,
      "percent": null,
      "status": "in_progress",
      "statusLabel": "In progress",
      "progressText": "25 questions answered",
      "stars": 1,
      "link": "/open/practice_list?filterType=subject&filterValue=Art%20%26%20Culture&examType=prelims",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "governance_society",
      "label": "Governance & Society",
      "topicsDone": 0,
      "topicsTotal": null,
      "percent": null,
      "status": "in_progress",
      "statusLabel": "In progress",
      "progressText": "25 questions answered",
      "stars": 1,
      "link": "/open/practice_list?filterType=subject&filterValue=Governance%20%26%20Society&examType=prelims",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "current_affairs",
      "label": "Current Affairs",
      "topicsDone": 0,
      "topicsTotal": null,
      "percent": null,
      "status": "in_progress",
      "statusLabel": "In progress",
      "progressText": "25 questions answered",
      "stars": 1,
      "link": "/open/practice_list?filterType=subject&filterValue=Current%20Affairs&examType=prelims",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "csat",
      "label": "CSAT",
      "topicsDone": 0,
      "topicsTotal": null,
      "percent": null,
      "status": "in_progress",
      "statusLabel": "In progress",
      "progressText": "25 questions answered",
      "stars": 1,
      "link": "/open/practice_list?filterType=subject&filterValue=CSAT&examType=prelims",
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    }
  ],
  "performance": {
    "range": "week",
    "label": "This week",
    "fromDate": "2026-11-10",
    "accuracyPercent": 59,
    "questionsAttempted": 242,
    "topicsCompleted": 8
  },
  "journal": {
    "latest": {
      "id": "177cd953-2e17-4555-9b9c-fe3375a2ef21",
      "dayNumber": 8,
      "date": "2026-11-16",
      "dateLabel": "Nov 16, 2026",
      "text": "Base Camp reached. Take a breath and enjoy the view; the next stretch starts whenever you are ready.",
      "author": "Monk",
      "band": 2
    },
    "total": 2
  },
  "prelims": {
    "promptDue": false,
    "canAnswer": false,
    "result": null,
    "answeredDate": null,
    "resultDate": null,
    "changeable": false,
    "nextAction": null,
    "prompt": {
      "title": "Did you clear Prelims?",
      "body": "The result is out. Let me know so I can set up the next phase of your Journey. Either way, your progress stays exactly where it is. Your Journey continues.",
      "yesLabel": "Yes, I cleared Prelims",
      "noLabel": "No, I didn't clear this time"
    }
  },
  "history": [],
  "serverDate": "2026-11-16"
}
```

### /stats Example C — exam without path rules

<!-- example: stats-response | rules v7 | exam without path rules -->
```json
{
  "available": false,
  "rulesVersion": null,
  "header": null,
  "stamps": [],
  "stars": null,
  "journey": [],
  "performance": null,
  "journal": null,
  "prelims": null,
  "history": [],
  "serverDate": "2026-11-16"
}
```

`/journal` for the same exam:

<!-- example: journal-response | rules v7 | exam without path rules -->
```json
{
  "available": false,
  "entries": [],
  "nextCursor": null,
  "total": 0,
  "serverDate": "2026-11-16"
}
```

---

## GET /activity-path/journal

Monk's journal, newest first (the "View full journal" screen). Backend item B4.

- **Auth / gate / exam / language / caching:** as `/path`.
- **Query:** `limit` 1–50 (default 20); `cursor` = the previous page's `nextCursor`. Anything else
  (`limit=0`, `limit=51`, `limit=ten`, a repeated param, a garbled or forged cursor) → `400`.
- **Side effects:** the **first page** (no cursor) runs the same catch-up pass as `/path`, so
  today's note and any backfill are written before listing. Later pages are plain reads.
- **Pagination:** keyset on (date, id), not offset. A note added while the learner scrolls
  never shifts or duplicates what was already listed. A newer note shows up on the next first-page
  load, and an older backfilled note shows up further down. Proven on real Postgres with a note
  inserted between pages.

| field | type | notes |
|---|---|---|
| `available` | boolean | `false` when the exam has no path rules: `entries` `[]`, `nextCursor` null, `total` 0 |
| `entries` | JournalEntry[] | newest first |
| `nextCursor` | string \| null | null on the last page |
| `total` | int | all notes the learner has in this exam |
| `serverDate` | string | today in IST |

**JournalEntry** — `id` string, `dayNumber` int (the Nth active day), `date` string (IST
`YYYY-MM-DD`), `dateLabel` string ("Nov 16, 2026" / "16 नवंबर 2026"), `text` string (the note in
the request language, exactly as written that day), `author` string ("Monk" in both languages since rules v9), `band`
int 0–4 (stable colour band = (dayNumber − 1) mod 5, so the app does not compute it). Sharing is
client-side.

### How notes are written (no AI)

- **One note per ACTIVE IST day**, keyed by the date and written once. A day counts as active if
  the learner studied in this exam that day; this is the same definition as active-day goals.
  Opening the app without studying writes nothing. `dayNumber` is that day's position among the
  learner's active days; for today it is exactly `/stats` `header.dayNumber`.
- **Both languages are rendered when the note is written** and stored. Switching language later
  shows the same note, and a later rule edit changes only future notes.
- **Stored under the lifetime attempt (0)**, so the journal survives a path restart.
- **Today:** written by the first call that sees study activity today, from what is recorded by
  then.
- **Backfill:** every active day from the path's first open up to yesterday that has no note gets
  one. Its numbers are rebuilt honestly from that day's timestamps: answers and accuracy from PYQ
  attempts, which are never deleted; library MCQs whose latest answer was that day; mocks submitted
  that day. A day where only reading or a Mains evaluation is recorded gets a generic note, never
  invented numbers. Oldest first, **at most 60 per call**; the rest continue on the next call.
  Proven: 131 missing days took three calls (61 → 121 → 131), and later calls added nothing.
- **History before the first open** becomes exactly **one** "story so far" note, keyed to the last
  of those days, with that period's active days, answers and topics. It is written only while the
  learner has no note at all.
- **Situation:** the first match in priority order wins. The table, placeholders and templates are in
  [docs/sme/ACTIVITY_PATH_RULES.md](../sme/ACTIVITY_PATH_RULES.md#monks-journal-notes); the
  variant rotates by day number.
- **Stage notes (B7):** "Begin Phase 2" writes a `stage_cleared` note and a restart writes a
  `stage_restart` note, as an **extra** entry for that day; the day's regular note is never replaced
  or lost. Stored key `<date>#stage_cleared.<attempt>` (or `#stage_restart.<attempt>`). The entry's
  `date` is still the plain IST day, and in the newest-first list it sits directly **above** that
  day's regular note (byte order, whatever the database collation). Day numbers continue across
  stage changes and restarts, because they are the learner's active days. A `nextCursor` that
  points at a stage note is valid. **Which wording (rules v8):** the first template whose numbers
  the learner has. Days on the path are named only from 30 active days, syllabus coverage only from
  50% (rule set `stageNotes`); below both, a plain number-free note ("You cleared Prelims. That is
  the gate that matters. The Summit is a different kind of climb, and we start it together."). No
  streak numbers. The result screen's `note` is the same text the write will store.

### /journal Example A — learner with history, first page (real response, en)

Two notes: the "story so far" for the 16 days before the path was opened (Nov 15), then today's
note (Day 17), written because Economy just earned its first star.

<!-- example: journal-response | rules v7 | learner with history (u_hist), first page, en -->
```json
{
  "available": true,
  "entries": [
    {
      "id": "0d0dab49-07ee-4408-8788-cc99959fcc10",
      "dayNumber": 17,
      "date": "2026-11-16",
      "dateLabel": "Nov 16, 2026",
      "text": "Economy now has its first star, Covered. Slow, steady work shows up like this.",
      "author": "Monk",
      "band": 1
    },
    {
      "id": "03ecaebf-542e-4c5d-aef9-1674bccc10bd",
      "dayNumber": 16,
      "date": "2026-11-15",
      "dateLabel": "Nov 15, 2026",
      "text": "Your story before you opened the path. Days studied: 16. Answers: 64. Topics completed: 2. Everything you built stays.",
      "author": "Monk",
      "band": 0
    }
  ],
  "nextCursor": null,
  "total": 2,
  "serverDate": "2026-11-16"
}
```

The same entry with `X-Lang: hi`:

<!-- example: journal-entry | rules v7 | the newest entry of the same learner, X-Lang: hi -->
```json
{
  "id": "0d0dab49-07ee-4408-8788-cc99959fcc10",
  "dayNumber": 17,
  "date": "2026-11-16",
  "dateLabel": "16 नवंबर 2026",
  "text": "अर्थव्यवस्था को अब अपना पहला सितारा मिल गया है: ‘कवर किया’। धीमी, स्थिर मेहनत ऐसे ही दिखाई देती है।",
  "author": "Monk",
  "band": 1
}
```

### /journal Example B — the veteran after its first open (real response, en, `limit=5`)

This is `/stats` Example B's learner. Its 2 notes, newest first: Day 8 `camp_completed` today
("Base Camp reached", written because Camp 1 was cleared) and, for Nov 15, the "story so far" note
for the 7 earlier days (7 days studied, 242 answers, 8 topics). With only 2 notes, `limit=5` returns
them all and `nextCursor` is null.

<!-- example: journal-response | rules v7 | the veteran, limit=5, en -->
```json
{
  "available": true,
  "entries": [
    {
      "id": "177cd953-2e17-4555-9b9c-fe3375a2ef21",
      "dayNumber": 8,
      "date": "2026-11-16",
      "dateLabel": "Nov 16, 2026",
      "text": "Base Camp reached. Take a breath and enjoy the view; the next stretch starts whenever you are ready.",
      "author": "Monk",
      "band": 2
    },
    {
      "id": "03601bdb-7eca-4310-ab49-4ef972714e1c",
      "dayNumber": 7,
      "date": "2026-11-15",
      "dateLabel": "Nov 15, 2026",
      "text": "You did not start from zero. Days studied: 7, answers: 242, topics finished: 8. The path starts from where you already are.",
      "author": "Monk",
      "band": 1
    }
  ],
  "nextCursor": null,
  "total": 2,
  "serverDate": "2026-11-16"
}
```

A learner with more than 5 notes would continue with `GET /activity-path/journal?limit=5&cursor=<nextCursor>`;
paging with a cursor lists every note exactly once.

---

## POST /activity-path/celebrations/seen

Marks celebrations as shown. This is the mobile `markCelebrationShown` seam. Backend item B4.

- **Auth / gate / exam:** as `/path` (the exam scopes which rows can be marked).
- **Body:** `{"ids": ["<celebration id>", …]}` with 1–50 UUIDs, taken from `/path`
  `celebrations[].id`. Missing or empty `ids`, more than 50, a non-UUID or a non-list → `400`. Any
  other top-level field → `400` (global validation pipe).
- **Effect:** stamps `seen_at` on the caller's **own unseen** celebration rows (camp / title / star)
  **in this exam**. Ids of another user, another exam, an already-seen row or a non-celebration row
  are ignored silently and counted in `ignored`. Idempotent: repeating a call marks nothing new.

| field | type | notes |
|---|---|---|
| `marked` | int | rows newly stamped seen |
| `ignored` | int | distinct ids not marked |

Real call (throwaway Postgres): the veteran's two celebration ids, plus one unknown id:

<!-- example: celebrations-seen-response | rules v7 | the veteran's 2 celebration ids + one unknown id -->
```json
{
  "marked": 2,
  "ignored": 1
}
```

Repeating the call returns `{"marked": 0, "ignored": 3}` (the two already-seen ids and the unknown
one), and the next `/path` call has `"celebrations": []`. Another learner's id is ignored the same
way and that learner's row stays unseen. Sending a valid own id with a different
`?exam=` returns `{"marked": 0, "ignored": 1}`.

---

## GET /activity-path/score

The "Overall score" screen: the overall percent, "Your four pillars" (View details), the subject
tabs and the selected subject's topic cards. Backend item B5.

- **Auth / gate / exam / language / caching:** exactly as `/path`.
- **Query:** `subject` = a subject key of the exam's rules (`polity`, `economy`, …, `csat`).
  Absent or empty → the **first subject in rule order with any activity** (a question answered, a
  document opened or progressed), else the first subject. A key the rules do not have (including a
  data spelling such as `Polity`) or a repeated `subject` →
  `400 {"success": false, "message": "subject must be one of: polity, economy, …"}`. The check runs
  before anything is read or written. If the exam has no rules the answer is `available: false`, not a 400.
- **Side effects — the same as `/path`.** `/score` runs the **same locked catch-up pass** (goals,
  steps, camps, stars, titles, notes, slot fills, the first-open rule). The score itself is
  computed on every call and never stored. `/score` then `/path` leaves exactly the ledger `/path`
  alone would, and so does running both at once (proven on real Postgres, see the end of this section).
  `/score` never returns celebrations.
- **Cost:** topics are listed for the selected subject only. On the throwaway seed one call took
  about 4–5 ms (`/path`: 3.7 ms). On staging, the four new statements take 1.4, 1.3, 6.7 and 0.6 ms.
  The 6.7 ms one is the PYQ-topic tally, a sequential scan of `pyq_papers`.

### Root

| field | type | notes |
|---|---|---|
| `available` | boolean | `false` when the exam has no path rules: then `overall`/`topics` are `null` and `pillars`/`subjects` are `[]` |
| `rulesVersion` | int \| null | |
| `overall` | {`percent`: int \| null, `label`: string} \| null | `label` "Overall score" / "कुल स्कोर". `percent` is the weighted mean of the **non-null** pillar percents by pillar `weight`, rounded down. It is computed from the **displayed** (already rounded) pillar percents, so 32/42/39/40 at 30/20/25/25 gives 37.75 → **37**. `null` when every pillar is null |
| `pillars` | Pillar[4] | always the four, in this order: `preparation`, `retention`, `consistency`, `performance` |
| `subjects` | ScoreSubject[] | one per subject, rule order (11 for UPSC) |
| `topics` | Topics \| null | the selected subject's topic list |
| `serverDate` | string | today in IST, `YYYY-MM-DD` |

**Rounding — one rule everywhere on this endpoint:** every percent is an integer **rounded down**
(the same as `/stats`).

**Pillar** — `key` (`preparation`\|`retention`\|`consistency`\|`performance`), `label` string,
`percent` int 0–100 \| **null**, `explanation` string (one line under the bar, from the rules).

A pillar is the weighted mean of its **signals**. A signal **with no data is left out**, and the
remaining weights are renormalised: a learner with no mock gets Performance = PYQ accuracy, not
half of it. A pillar whose signals all have no data is **null**, never 0. For a learner with
**no active day in this exam** (brand new, or active only in another exam), every signal has no
data, so all four pillars and `overall.percent` are `null`. For an active learner, the count
signals keep a real 0: "0 of 120 documents covered" is information.

| signal | value 0–100 (exact meaning) | source | no data when |
|---|---|---|---|
| `topic_coverage` | roadmap completion over **every mapped subject's** documents together: Σ(completed ? 1 : answered ÷ questions) ÷ documents. Partial reading counts, the same as the Library's % and `/stats` `journey[].percent`. Folders with no path subject (Ethics) are not counted | `content_documents` + `user_document_progress`, the roadmap's language per folder | learner never active, or the exam has no documents in mapped folders |
| `questions_attempted` | min(100, distinct questions answered ÷ `target`). Counts PYQ and library MCQs, latest answer per question, every subject (= `/path` "Attempt N questions") | `pyq_attempts`, `user_question_attempts` | learner never active |
| `retry_accuracy` | correct ÷ answered, **scored by the latest answer**, over questions re-answered on a **later IST day** than first answered ("did the second look stick?"). PYQ: every attempt is stored. Library: only the latest re-answer survives | `pyq_attempts` (window per question), `user_question_attempts` `created_at`/`updated_at` | nothing re-answered on a later day |
| `recency` | mean over the subjects the learner has **touched** of max(0, 1 − days since last touch ÷ `target`). Touched today = 100; `target` (14) or more days ago = 0. A touch is a PYQ answered, a library MCQ answered, or a document opened, progressed or completed, the same events as an active day. Untouched subjects are not averaged in, because coverage measures breadth | per-subject last IST day of those events, mapped through `pyqSubject` / `roadmapSubjects` | no subject ever touched |
| `active_days_ratio` | active IST days in the last ⌈`target`⌉ days (today included) ÷ ⌈`target`⌉. **Not** the streak | the path's active days (see "Semantics") | learner never active |
| `streak` | min(100, live streak ÷ `target`). **Unused by the defaults** (owner ruling: no streak dependency); kept for overrides | `user_auth` streak, alive only if credited today/yesterday IST | learner never active |
| `pyq_accuracy` | correct ÷ answered, latest answer per PYQ, **every subject** (mapped or not), playable questions only. Library MCQs are **not** counted | `pyq_attempts` ⨝ `pyq_papers` | no PYQ answered |
| `mock_score` | the **mean** accuracy of **every submitted** mock attempt in the exam. Unsubmitted attempts are not counted. (`/path`'s mock goal uses the best attempt; a pillar should reflect typical performance) | `simulation_attempts` (accuracy is a 0–1 fraction) | no mock submitted |
| `mains_evaluation_score` | mean of score ÷ marks over the **latest** evaluation per Mains question (marks > 0). Counted whenever such data exists, at any stage; unused by the defaults | `mains_evaluations` | no Mains answer evaluated |

Default pillars (rules v5, owner to confirm the numbers):

| pillar | weight | inputs | explanation (en) |
|---|---|---|---|
| Preparation | 30 | `topic_coverage` 60, `questions_attempted` 40 (target 3000) | Study material covered and questions attempted |
| Retention | 20 | `retry_accuracy` 60, `recency` 40 (target 14 days) | Accuracy when you re-answer questions on a later day, and how recently you practised each subject |
| Consistency | 25 | `active_days_ratio` 100 (target 30 days) | Days you studied in the last 30 days |
| Performance | 25 | `pyq_accuracy` 50, `mock_score` 50 | Your PYQ accuracy and your mock test scores |

**ScoreSubject** — `key`, `label` string; `stars` int 0–3 (= `/stats` `stars.subjects`, recorded
in this same call); `topicsDone` int, `topicsTotal` int (0 when the subject lists no topics;
never null here); `selected` boolean (exactly one is true). For a subject whose topics are
documents, these equal `/stats` `journey[].topicsDone`/`topicsTotal`, because they count the same
documents. For the selected subject they equal `topics.done`/`topics.total`. `confidence`
Confidence (B6): the subject-level judge, the same object as `/stats` `journey[].confidence`.

**Topics**

| field | type | meaning |
|---|---|---|
| `subjectKey` | string | the selected subject |
| `model` | `"documents"\|"pyq_topics"\|"none"` | what a topic is for this subject, see below |
| `infoText` | string | the info-icon line, for example "Topics are the study documents for Economy. Accuracy comes from each document's practice questions." |
| `done` | int | topics with `status: "completed"` |
| `total` | int | topics listed (the design's "Topics (18/34)" is `done`/`total`) |
| `items` | Topic[] | never null; `[]` when the subject has none |

**What a "topic" is.** This was decided from the data on 2026-11-16:
- **A subject with study material** (mapped folders that hold documents in this exam): its topics
  are its **study documents**. They are listed in the Library roadmap's language per folder: the
  requested language if the folder has any document in it, else English. A Hindi learner whose
  folder has Hindi documents therefore sees only those. This is the unit `/stats` and the path
  already count as topics. Order: folder (rule order), then Library section, then title.
  `model: "documents"`.
- **A subject without study material** (Art & Culture, Governance & Society, Current Affairs, CSAT;
  also a mapped subject whose folder is empty): its topics are the **PYQ `topic` strings** under its
  PYQ subject, ordered A–Z. The question universe follows the PYQ list's language rule (the
  requested language if the subject has questions in it, else English). Answers count in **any**
  language, because topic keys are English in every language row. A topic is listed when it has
  questions in the served language or answers in any language. Labels are those English strings,
  also for Hindi. `model: "pyq_topics"`.
- **Not joined:** PYQ topic strings and study documents. On staging, 0–10 of 86–193 PYQ topics per
  subject exactly match a document title, and none match a Library section. Fuzzy matching would
  file answers under the wrong card. So a document's accuracy comes from its **library MCQs only**,
  and a mapped subject's PYQ answers show in its stars and the pillars, not on its topic cards.

**Topic**

| field | type | meaning |
|---|---|---|
| `key` | string | `doc:<document id>` or `pyq:<PYQ topic>`; stable |
| `kind` | `"document"\|"pyq_topic"` | |
| `label` | string | document title / PYQ topic |
| `group` | string \| null | the document's Library section (`content_documents.topic`, e.g. "Economy (Prelims)"). It tells apart same-titled documents ("UPSC Syllabus" appears in several sections). null for PYQ topics |
| `status` | `"not_started"\|"in_progress"\|"completed"` | document: `completed` = completed in the Library; `in_progress` = opened or any MCQ answered. PYQ topic: `completed` = every served-language question answered; `in_progress` = any answer |
| `statusLabel` | string | as `/stats` ("Not started" / "In progress" / "Completed"; hi "शुरू नहीं हुआ" / "जारी है" / "पूरा हुआ") |
| `accuracyPercent` | int \| null | correct ÷ answers, rounded down; **null when nothing answered** |
| `answers` | int | questions answered, latest answer per question. Document: its live library MCQs. PYQ topic: playable PYQs, any language |
| `progressPercent` | int \| null | document: 100 when completed, else questions answered ÷ its questions (the Library's per-document number); null for an uncompleted document with no questions. PYQ topic: served-language questions answered ÷ its questions; null when it has none in that language |
| `tag` | `"weak"` \| null | `weak` when accuracy is **strictly below** `topics.weakBelowPercent` (default 40) over **at least** `topics.minAnswers` answers (default 5). Never on fewer answers |
| `tagLabel` | string \| null | "Weak" / "कमज़ोर"; null when `tag` is null |
| `confidence` | Confidence | **B6.** A document is judged on two signals: its progress % against its MCQ accuracy, and days since it was last touched. The judge's floor is 20 MCQ answers on the document. A PYQ topic is never judged: it holds only about 1–3 questions, so it always reads `unknown`. See [Confidence judge](#confidence-judge-b6) |
| `link` | string | document: `/open/doc/<id>`. PYQ topic: `/open/replay_session?subject=<PYQ subject>&topic=<topic>` while one of its latest answers is wrong (the replay feed's own rule); otherwise `/open/practice_list?filterType=subject&filterValue=<PYQ subject>&examType=prelims`. There is no per-topic PYQ list link: `practice_list` filters only by year or subject |

### /score Example A — brand-new learner (real response, throwaway Postgres, en, no `subject`)

There is no activity, so every pillar and the overall score are `null`, not 0. No subject has
activity, so the first one (Polity) is selected. Polity has 3 study documents, none started.

<!-- example: score-response | rules v7 | brand-new learner, en, no subject -->
```json
{
  "available": true,
  "rulesVersion": 7,
  "overall": {
    "percent": null,
    "label": "Overall score"
  },
  "pillars": [
    {
      "key": "preparation",
      "label": "Preparation",
      "percent": null,
      "explanation": "Study material covered and questions attempted"
    },
    {
      "key": "retention",
      "label": "Retention",
      "percent": null,
      "explanation": "Accuracy when you re-answer questions on a later day, and how recently you practised each subject"
    },
    {
      "key": "consistency",
      "label": "Consistency",
      "percent": null,
      "explanation": "Days you studied in the last 30 days"
    },
    {
      "key": "performance",
      "label": "Performance",
      "percent": null,
      "explanation": "Your PYQ accuracy and your mock test scores"
    }
  ],
  "subjects": [
    {
      "key": "polity",
      "label": "Polity",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 3,
      "selected": true,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "economy",
      "label": "Economy",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 5,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "history",
      "label": "History",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "geography",
      "label": "Geography",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "environment",
      "label": "Environment & Ecology",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "science_tech",
      "label": "Science & Technology",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "ir_security",
      "label": "IR & Security",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 2,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "art_culture",
      "label": "Art & Culture",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 4,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "governance_society",
      "label": "Governance & Society",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "current_affairs",
      "label": "Current Affairs",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "csat",
      "label": "CSAT",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    }
  ],
  "topics": {
    "subjectKey": "polity",
    "model": "documents",
    "infoText": "Topics are the study documents for Polity. Accuracy comes from each document's practice questions.",
    "done": 0,
    "total": 3,
    "items": [
      {
        "key": "doc:p2",
        "kind": "document",
        "label": "Elections",
        "group": "Contemporary Issues: Governance (Mains)",
        "status": "not_started",
        "statusLabel": "Not started",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 0,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/p2"
      },
      {
        "key": "doc:p1",
        "kind": "document",
        "label": "Citizenship",
        "group": "Indian Polity And Governance (Prelims)",
        "status": "not_started",
        "statusLabel": "Not started",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 0,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/p1"
      },
      {
        "key": "doc:b7doc-1",
        "kind": "document",
        "label": "Doc 1",
        "group": "T",
        "status": "not_started",
        "statusLabel": "Not started",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 0,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/b7doc-1"
      }
    ]
  },
  "serverDate": "2026-11-16"
}
```

A learner whose history is entirely in **another exam** gets the same nulls in `upsc-cse` (proven
with the same seed).

### /score Example B — learner with history, `?subject=economy` (real response, en)

> Recorded under rules v7, so it carries the B6 `confidence` objects: Economy is `clear`, and the
> topic "Government Schemes" is a `gap` ("Confidence is way higher than accuracy."). The other
> topics are `unknown`. The same learner without `subject` is
> [/score Example B6](#score-example-b6--the-same-learner-under-the-v1-judge-real-response-en).

The seed:
- Economy PYQs: 10 answered wrong 20 days ago, then re-answered 2 days ago (8 right); 20 more
  answered today (15 right).
- Art & Culture PYQs 5 days ago: 11 answered, 6 right, including one Hindi question each in
  "Religions in India" and "Bhakti".
- Economy library MCQs: "Government Schemes" 20 answered, 7 right (4 of them re-answered on a later
  day, 2 right); "Banking" completed, 9 of 10 right; "Basic Concepts" 3 of 8 answered.
- One Polity document completed 25 days ago.
- 2 submitted mocks (40%, 60%) and 2 Mains evaluations.
- Noise that must not count: another exam's PYQ, document and mock; a board-deleted PYQ; a PYQ with
  no answer key; a draft document; an inactive library question; an unsubmitted mock.

How each pillar comes out:

| pillar | working | percent |
|---|---|---|
| Preparation | `topic_coverage` (2.375 of 5 Economy documents + 1 of 3 Polity documents + 0 of the 6 documents in the other mapped subjects) ÷ 14 = 24.1 at 60%, plus `questions_attempted` 74 ÷ 3000 = 2.47 at 40% → 15.45 | **15** |
| Retention | `retry_accuracy` 10 of 14 = 71.4 at 60%, plus `recency` (Economy today 100, Art & Culture 1 day ago 92.9, Polity 25 days ago 0) ÷ 3 = 64.3 at 40% → 68.57 | **68** |
| Consistency | 17 active days in the last 30 → 56.7 | **56** |
| Performance | `pyq_accuracy` 29 of 41 = 70.7, plus `mock_score` mean 50 → 60.4 | **60** |
| Overall | (15×30 + 68×20 + 56×25 + 60×25) ÷ 100 = 47.1 | **47** |

The Art & Culture touch "1 day ago" comes from the board-deleted / no-key noise attempt. A touch is
an active-day event, and those count the same way in active days; they add no answers or accuracy
anywhere. "Government Schemes" is **Weak**: 35% over 20 answers. "Basic Concepts" is not: 100%.

<!-- example: score-response | rules v7 | learner with history (u_hist), ?subject=economy, en — weak topic, judge -->
```json
{
  "available": true,
  "rulesVersion": 7,
  "overall": {
    "percent": 47,
    "label": "Overall score"
  },
  "pillars": [
    {
      "key": "preparation",
      "label": "Preparation",
      "percent": 15,
      "explanation": "Study material covered and questions attempted"
    },
    {
      "key": "retention",
      "label": "Retention",
      "percent": 68,
      "explanation": "Accuracy when you re-answer questions on a later day, and how recently you practised each subject"
    },
    {
      "key": "consistency",
      "label": "Consistency",
      "percent": 56,
      "explanation": "Days you studied in the last 30 days"
    },
    {
      "key": "performance",
      "label": "Performance",
      "percent": 60,
      "explanation": "Your PYQ accuracy and your mock test scores"
    }
  ],
  "subjects": [
    {
      "key": "polity",
      "label": "Polity",
      "stars": 0,
      "topicsDone": 1,
      "topicsTotal": 3,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "economy",
      "label": "Economy",
      "stars": 1,
      "topicsDone": 1,
      "topicsTotal": 5,
      "selected": true,
      "confidence": {
        "state": "clear",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "history",
      "label": "History",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "geography",
      "label": "Geography",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "environment",
      "label": "Environment & Ecology",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "science_tech",
      "label": "Science & Technology",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "ir_security",
      "label": "IR & Security",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 2,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "art_culture",
      "label": "Art & Culture",
      "stars": 1,
      "topicsDone": 1,
      "topicsTotal": 5,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "governance_society",
      "label": "Governance & Society",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "current_affairs",
      "label": "Current Affairs",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "csat",
      "label": "CSAT",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    }
  ],
  "topics": {
    "subjectKey": "economy",
    "model": "documents",
    "infoText": "Topics are the study documents for Economy. Accuracy comes from each document's practice questions.",
    "done": 1,
    "total": 5,
    "items": [
      {
        "key": "doc:e3",
        "kind": "document",
        "label": "Basic Concepts",
        "group": "Economy (Prelims)",
        "status": "in_progress",
        "statusLabel": "In progress",
        "accuracyPercent": 100,
        "answers": 3,
        "progressPercent": 37,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/e3"
      },
      {
        "key": "doc:e4",
        "kind": "document",
        "label": "Public Finance in India",
        "group": "Economy (Prelims)",
        "status": "not_started",
        "statusLabel": "Not started",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 0,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/e4"
      },
      {
        "key": "doc:e2",
        "kind": "document",
        "label": "Banking",
        "group": "Policy & Reforms In Indian Economy (Mains)",
        "status": "completed",
        "statusLabel": "Completed",
        "accuracyPercent": 90,
        "answers": 10,
        "progressPercent": 100,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/e2"
      },
      {
        "key": "doc:e1",
        "kind": "document",
        "label": "Government Schemes",
        "group": "Policy & Reforms In Indian Economy (Mains)",
        "status": "in_progress",
        "statusLabel": "In progress",
        "accuracyPercent": 35,
        "answers": 20,
        "progressPercent": 100,
        "tag": "weak",
        "tagLabel": "Weak",
        "confidence": {
          "state": "gap",
          "message": "Confidence is way higher than accuracy.",
          "actionLabel": "Take reality check quiz",
          "link": "/open/replay_session?subject=Economy&topic=Policy%20%26%20Reforms%20In%20Indian%20Economy%20(Mains)"
        },
        "link": "/open/doc/e1"
      },
      {
        "key": "doc:b7doc-2",
        "kind": "document",
        "label": "Doc 2",
        "group": "T",
        "status": "not_started",
        "statusLabel": "Not started",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 0,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/b7doc-2"
      }
    ]
  },
  "serverDate": "2026-11-16"
}
```

Without `subject` the same learner gets Polity: it comes first in rule order and has activity (the
completed document).

### /score Example C — an unmapped subject, `?subject=art_culture` (real response; only `topics` shown, the rest equals Example B with `selected` moved)

English:

<!-- example: score-response | rules v7 | the same learner, unmapped subject ?subject=art_culture, en -->
```json
{
  "available": true,
  "rulesVersion": 7,
  "overall": {
    "percent": 47,
    "label": "Overall score"
  },
  "pillars": [
    {
      "key": "preparation",
      "label": "Preparation",
      "percent": 15,
      "explanation": "Study material covered and questions attempted"
    },
    {
      "key": "retention",
      "label": "Retention",
      "percent": 68,
      "explanation": "Accuracy when you re-answer questions on a later day, and how recently you practised each subject"
    },
    {
      "key": "consistency",
      "label": "Consistency",
      "percent": 56,
      "explanation": "Days you studied in the last 30 days"
    },
    {
      "key": "performance",
      "label": "Performance",
      "percent": 60,
      "explanation": "Your PYQ accuracy and your mock test scores"
    }
  ],
  "subjects": [
    {
      "key": "polity",
      "label": "Polity",
      "stars": 0,
      "topicsDone": 1,
      "topicsTotal": 3,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "economy",
      "label": "Economy",
      "stars": 1,
      "topicsDone": 1,
      "topicsTotal": 5,
      "selected": false,
      "confidence": {
        "state": "clear",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "history",
      "label": "History",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "geography",
      "label": "Geography",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "environment",
      "label": "Environment & Ecology",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "science_tech",
      "label": "Science & Technology",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "ir_security",
      "label": "IR & Security",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 2,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "art_culture",
      "label": "Art & Culture",
      "stars": 1,
      "topicsDone": 1,
      "topicsTotal": 5,
      "selected": true,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "governance_society",
      "label": "Governance & Society",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "current_affairs",
      "label": "Current Affairs",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "csat",
      "label": "CSAT",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    }
  ],
  "topics": {
    "subjectKey": "art_culture",
    "model": "pyq_topics",
    "infoText": "Topics come from previous-year questions in Art & Culture. Accuracy uses your latest answer to each question.",
    "done": 1,
    "total": 5,
    "items": [
      {
        "key": "pyq:Bhakti",
        "kind": "pyq_topic",
        "label": "Bhakti",
        "group": null,
        "status": "in_progress",
        "statusLabel": "In progress",
        "accuracyPercent": 0,
        "answers": 1,
        "progressPercent": null,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/replay_session?subject=Art%20%26%20Culture&topic=Bhakti"
      },
      {
        "key": "pyq:Indian Music",
        "kind": "pyq_topic",
        "label": "Indian Music",
        "group": null,
        "status": "not_started",
        "statusLabel": "Not started",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 0,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/practice_list?filterType=subject&filterValue=Art%20%26%20Culture&examType=prelims"
      },
      {
        "key": "pyq:Religions in India",
        "kind": "pyq_topic",
        "label": "Religions in India",
        "group": null,
        "status": "in_progress",
        "statusLabel": "In progress",
        "accuracyPercent": 42,
        "answers": 7,
        "progressPercent": 85,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/replay_session?subject=Art%20%26%20Culture&topic=Religions%20in%20India"
      },
      {
        "key": "pyq:T",
        "kind": "pyq_topic",
        "label": "T",
        "group": null,
        "status": "not_started",
        "statusLabel": "Not started",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 0,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/practice_list?filterType=subject&filterValue=Art%20%26%20Culture&examType=prelims"
      },
      {
        "key": "pyq:Temple Architecture",
        "kind": "pyq_topic",
        "label": "Temple Architecture",
        "group": null,
        "status": "completed",
        "statusLabel": "Completed",
        "accuracyPercent": 100,
        "answers": 3,
        "progressPercent": 100,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/practice_list?filterType=subject&filterValue=Art%20%26%20Culture&examType=prelims"
      }
    ]
  },
  "serverDate": "2026-11-16"
}
```

"Bhakti" exists only in Hindi, but it is listed because it was answered. "Religions in India" has 6
English answers and 1 Hindi answer (42%), which is above the 40% Weak line.

Hindi, the same learner: the subject has Hindi questions, so the universe is Hindi. "Indian Music"
has neither Hindi questions nor answers, so it drops out. "Temple Architecture" was completed in
English but has no Hindi questions, so it shows `in_progress` with `progressPercent: null`.

<!-- example: score-response | rules v7 | the same, ?subject=art_culture, X-Lang: hi -->
```json
{
  "available": true,
  "rulesVersion": 7,
  "overall": {
    "percent": 44,
    "label": "कुल स्कोर"
  },
  "pillars": [
    {
      "key": "preparation",
      "label": "तैयारी",
      "percent": 6,
      "explanation": "पढ़ी गई अध्ययन सामग्री और हल किए गए सवाल"
    },
    {
      "key": "retention",
      "label": "स्मरण",
      "percent": 68,
      "explanation": "बाद के दिन दोबारा हल किए गए सवालों में सटीकता, और हर विषय का अभ्यास कितने समय पहले किया"
    },
    {
      "key": "consistency",
      "label": "निरंतरता",
      "percent": 56,
      "explanation": "पिछले 30 दिनों में आपने कितने दिन पढ़ाई की"
    },
    {
      "key": "performance",
      "label": "प्रदर्शन",
      "percent": 60,
      "explanation": "PYQ में आपकी सटीकता और मॉक टेस्ट के अंक"
    }
  ],
  "subjects": [
    {
      "key": "polity",
      "label": "राजव्यवस्था",
      "stars": 0,
      "topicsDone": 1,
      "topicsTotal": 3,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "economy",
      "label": "अर्थव्यवस्था",
      "stars": 1,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "clear",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "history",
      "label": "इतिहास",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "geography",
      "label": "भूगोल",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "environment",
      "label": "पर्यावरण एवं पारिस्थितिकी",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "science_tech",
      "label": "विज्ञान एवं प्रौद्योगिकी",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "ir_security",
      "label": "अंतरराष्ट्रीय संबंध एवं सुरक्षा",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 2,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "art_culture",
      "label": "कला एवं संस्कृति",
      "stars": 1,
      "topicsDone": 1,
      "topicsTotal": 3,
      "selected": true,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "governance_society",
      "label": "शासन एवं समाज",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "current_affairs",
      "label": "समसामयिकी",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "csat",
      "label": "सीसैट",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    }
  ],
  "topics": {
    "subjectKey": "art_culture",
    "model": "pyq_topics",
    "infoText": "टॉपिक कला एवं संस्कृति के पिछले वर्षों के सवालों (PYQ) से हैं। सटीकता हर सवाल के आपके नवीनतम उत्तर से है।",
    "done": 1,
    "total": 3,
    "items": [
      {
        "key": "pyq:Bhakti",
        "kind": "pyq_topic",
        "label": "Bhakti",
        "group": null,
        "status": "completed",
        "statusLabel": "पूरा हुआ",
        "accuracyPercent": 0,
        "answers": 1,
        "progressPercent": 100,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/replay_session?subject=Art%20%26%20Culture&topic=Bhakti"
      },
      {
        "key": "pyq:Religions in India",
        "kind": "pyq_topic",
        "label": "Religions in India",
        "group": null,
        "status": "in_progress",
        "statusLabel": "जारी है",
        "accuracyPercent": 42,
        "answers": 7,
        "progressPercent": 25,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/replay_session?subject=Art%20%26%20Culture&topic=Religions%20in%20India"
      },
      {
        "key": "pyq:Temple Architecture",
        "kind": "pyq_topic",
        "label": "Temple Architecture",
        "group": null,
        "status": "in_progress",
        "statusLabel": "जारी है",
        "accuracyPercent": 100,
        "answers": 3,
        "progressPercent": null,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/practice_list?filterType=subject&filterValue=Art%20%26%20Culture&examType=prelims"
      }
    ]
  },
  "serverDate": "2026-11-16"
}
```

### /score Example D — exam without path rules

<!-- example: score-response | rules v7 | exam without path rules -->
```json
{
  "available": false,
  "rulesVersion": null,
  "overall": null,
  "pillars": [],
  "subjects": [],
  "topics": null,
  "serverDate": "2026-11-16"
}
```

### /score — what was proven on real Postgres (throwaway `postgres:17-alpine`, full migration history)

- `/score` then `/path` (learner A), `/path` alone (twin B) and `/score` ∥ `/path` concurrently
  (twin C) end in **identical** ledgers (6 rows each) and progress rows, with no duplicate keys.
  Three more `/score` calls changed nothing.
- Each `/score` tab's `stars` equals `/stats` `stars.subjects`. Its `topicsDone`/`topicsTotal`
  equals `/stats` `journey` for every subject with study material.
- The noise rows changed no count, accuracy or mean. The one exception is the board-deleted /
  no-key attempt, which counts as a touch, exactly as it counts as an active day.

### /score Example B6 — the same learner under the v1 judge (real response, en)

Example B's learner without `subject`, so the default selection applies: Polity is selected (first
in rule order with activity, its completed document "Citizenship"). Every subject carries the B6
`confidence` object. Economy is `clear` (`message`, `actionLabel` and `link` are null) and
every other subject is `unknown`, since it has fewer than 20 answers. The Polity topics are all
`unknown` too: a document is judged only after 20 MCQ answers. For a **gap** verdict, see the
"Government Schemes" topic in Example B (100% progress against 35% accuracy over 20 answers); the
signals behind a verdict are in [Confidence judge](#confidence-judge-b6).

<!-- example: score-response | rules v7 | learner with history, no subject (default selection), en -->
```json
{
  "available": true,
  "rulesVersion": 7,
  "overall": {
    "percent": 47,
    "label": "Overall score"
  },
  "pillars": [
    {
      "key": "preparation",
      "label": "Preparation",
      "percent": 15,
      "explanation": "Study material covered and questions attempted"
    },
    {
      "key": "retention",
      "label": "Retention",
      "percent": 68,
      "explanation": "Accuracy when you re-answer questions on a later day, and how recently you practised each subject"
    },
    {
      "key": "consistency",
      "label": "Consistency",
      "percent": 56,
      "explanation": "Days you studied in the last 30 days"
    },
    {
      "key": "performance",
      "label": "Performance",
      "percent": 60,
      "explanation": "Your PYQ accuracy and your mock test scores"
    }
  ],
  "subjects": [
    {
      "key": "polity",
      "label": "Polity",
      "stars": 0,
      "topicsDone": 1,
      "topicsTotal": 3,
      "selected": true,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "economy",
      "label": "Economy",
      "stars": 1,
      "topicsDone": 1,
      "topicsTotal": 5,
      "selected": false,
      "confidence": {
        "state": "clear",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "history",
      "label": "History",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "geography",
      "label": "Geography",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "environment",
      "label": "Environment & Ecology",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "science_tech",
      "label": "Science & Technology",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "ir_security",
      "label": "IR & Security",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 2,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "art_culture",
      "label": "Art & Culture",
      "stars": 1,
      "topicsDone": 1,
      "topicsTotal": 5,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "governance_society",
      "label": "Governance & Society",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "current_affairs",
      "label": "Current Affairs",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    },
    {
      "key": "csat",
      "label": "CSAT",
      "stars": 0,
      "topicsDone": 0,
      "topicsTotal": 1,
      "selected": false,
      "confidence": {
        "state": "unknown",
        "message": null,
        "actionLabel": null,
        "link": null
      }
    }
  ],
  "topics": {
    "subjectKey": "polity",
    "model": "documents",
    "infoText": "Topics are the study documents for Polity. Accuracy comes from each document's practice questions.",
    "done": 1,
    "total": 3,
    "items": [
      {
        "key": "doc:p2",
        "kind": "document",
        "label": "Elections",
        "group": "Contemporary Issues: Governance (Mains)",
        "status": "not_started",
        "statusLabel": "Not started",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 0,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/p2"
      },
      {
        "key": "doc:p1",
        "kind": "document",
        "label": "Citizenship",
        "group": "Indian Polity And Governance (Prelims)",
        "status": "completed",
        "statusLabel": "Completed",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 100,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/p1"
      },
      {
        "key": "doc:b7doc-1",
        "kind": "document",
        "label": "Doc 1",
        "group": "T",
        "status": "not_started",
        "statusLabel": "Not started",
        "accuracyPercent": null,
        "answers": 0,
        "progressPercent": 0,
        "tag": null,
        "tagLabel": null,
        "confidence": {
          "state": "unknown",
          "message": null,
          "actionLabel": null,
          "link": null
        },
        "link": "/open/doc/b7doc-1"
      }
    ]
  },
  "serverDate": "2026-11-16"
}
```

This body holds no `gap`: Economy is `clear`. In Hindi a gap banner reads
`"आत्मविश्वास सटीकता से कहीं ज़्यादा है।"` and the button `"रियलिटी चेक क्विज़ लें"`.
`/stats` `journey[economy].confidence` is the same object as the subject's `confidence` here.

---

## Confidence judge (B6)

**What it means, in plain words:** the app never asks the learner how confident they feel; there
are no confidence chips. Instead the backend looks at **behaviour** for signs that the learner
believes they are further along than their answers show. Examples: material marked as covered but
low accuracy; wrong answers given very fast; doing worse in a timed mock than in practice;
skipping questions in mocks; getting a question wrong again after re-trying it; leaving a subject
untouched for weeks. When enough of those point the same way, the subject (or study document) is
a **gap**. The app then shows the banner "Confidence is way higher than accuracy." with the button
"Take reality check quiz", which opens the replay quiz of that subject's mistakes.

**The Confidence object** (on `/score` `subjects[]`, `/score` `topics.items[]` and `/stats` `journey[]`):

| field | type | meaning |
|---|---|---|
| `state` | `"gap"\|"clear"\|"unknown"` | `gap` = show the banner; `clear` = judged, no gap; `unknown` = not judged (too few answers, or no evidence yet) |
| `message` | string \| null | banner text, **only for `gap`**; from the rules (`confidenceJudge.banner`) |
| `actionLabel` | string \| null | button text, **only for `gap`** |
| `link` | string \| null | the reality-check quiz, **only for `gap`**: subject → `/open/replay_session?subject=<PYQ subject>` (no PYQ spelling: the first study folder); document → `/open/replay_session?subject=<folder>&topic=<Library section>` (the replay feed groups library mistakes by exactly that pair, so the quiz covers the section the document is in). The app resolves it to ReplaySessionRoute(subject, topic?) |

**How a subject is judged.** Each of six signals either fires, does not fire, or has **no data**.
A signal with no data is left out, and the weights of the rest are scaled up to fill the gap. It
is never guessed.

| signal (rule key) | value | fires at (default) | weight | no data when |
|---|---|---|---|---|
| `coverageAccuracyGap` | study-material coverage % − accuracy % (PYQ + library, latest answer) | ≥ 25 points | 25 | no study material mapped / no documents |
| `fastAndWrong` | of the WRONG PYQ answers with a measured time, the share given in under `fastSeconds` (15 s) | ≥ 20% | 20 | no PYQ answer with a measured time yet. Every answer before the app sends `timeSpentSeconds` is unknown, never "fast". Library MCQs have no time |
| `timedVsPractice` | practice accuracy − accuracy on the subject's mock questions (answered ones) | ≥ 15 points | 15 | the subject has no `mockSubjects`, or fewer than 10 answered mock questions in it |
| `skipPattern` | unanswered ÷ presented mock questions of the subject, in submitted mocks | ≥ 20% | 10 | no `mockSubjects`, or fewer than 10 mock questions presented |
| `wrongOnRetry` | of the PYQs answered 2+ times, the share whose LATEST answer is still wrong | ≥ 30% | 20 | no PYQ re-attempted. Library MCQs keep only their last answer, so they are excluded |
| `staleness` | days since the subject was last touched (PYQ, library MCQ, document) | ≥ 21 days | 10 | never touched |

Then:
- **Floor:** with fewer than **20** answers in the subject, nothing is judged (`unknown`).
- **Evidence first:** the four signals that compare belief with answers (coverage gap,
  fast-and-wrong, timed-vs-practice, wrong-on-retry) are the evidence. If none of them has data,
  the state is `unknown`. Skip pattern and staleness can strengthen a gap but never create one alone.
- **Score** = weight of the fired signals ÷ weight of the signals with data × 100.
- **`gap`** when score ≥ **50** AND at least one evidence signal fired; otherwise `clear`.
- **Cleared** (what `confidence_gap_cleared` goals and the Mastered star need) = `clear` AND
  score ≤ **30**. Between 30 and 50 no banner shows, but the goal is not met yet (hysteresis).
- **Mock subjects:** mock questions use a third subject vocabulary ("Economics", "Science and
  Technology"). Only unambiguous pairs are mapped (see the SME doc). Where a subject has no mapping,
  the two mock signals are **excluded** for it rather than compared exam-wide. An exam-wide mock
  score set against one subject's practice accuracy would flag a strong Polity learner for weak
  mocks overall.

**Already earned stays earned:** a `confidence_gap_cleared` goal or a Mastered star recorded under
the old v0 judge (or earlier on the same data) is never taken back when v1 now says gap.

**Sanity on real data (read-only, 2026-11-16):**
- No staging learner reaches the floor in any subject: 7 learners, 77 subjects, all `unknown`.
- The 25 most active **production** learners give 275 subjects: 133 `clear`, 128 `unknown`
  (below the floor), 14 `gap`.
- Every production gap comes from wrong-on-retry plus staleness, in subjects with 39–79% accuracy.
- Strong learners (90–100% accuracy) are all `clear`.

The weights are the B1 numbers, so tune them with the SME judge read
(`GET /sme/activity-path/judge/:examId/:userId`).

---

## POST /pyq/:id/submit and POST /pyq/wrong-answers/:questionId/retry — answer time (B6)

Both bodies accept one more **optional** field:

| field | type | rule |
|---|---|---|
| `timeSpentSeconds` | int \| null | whole seconds the learner spent on the question before answering; 0–3600. Absent or `null` means unknown. A negative, fractional, over-an-hour or non-numeric value → `400` |

- It is **stored only**, in `pyq_attempts.time_spent_seconds`. Grading, the attempt number,
  every response field and the Neo4j mirror are unchanged; the mirror still gets 0.
- When the field is absent, the row now stores NULL instead of the old hard-coded 0. Both mean
  unknown. No reader of the column assumed a number.
- A library retry has no time column and ignores it. The fast-and-wrong signal uses only answers
  with time > 0.

**⚠️ DEPLOY ORDER — BACKEND FIRST.** The global validation pipe rejects unknown fields. An app
build that sends `timeSpentSeconds` to a backend **without** this change gets a **400 on every
answer submit**. Deploy the backend (commit "accept optional timeSpentSeconds on answer submit")
to production **before** any app build that sends the field goes out.

## Prelims result and the stage change (B7)

"Did you clear Prelims?" → "You made it through" + "Begin Phase 2", or "Let's see where you really
stand" + calibration test + restart. UPSC only (any exam whose rules have a `prelims` first stage).
Auth, gate (`dashboard.activityPath` → `403`), exam, `X-Lang` and `Vary` exactly as `/path`. Every
string is final; every key is always present.

### The state machine

| state (per learner and exam) | how it is entered | can it be undone? |
|---|---|---|
| Prelims stage, no answer | first open; or a restart | — |
| answered `cleared` / `not_cleared` (Prelims stage, `prelims.changeable: true`) | `POST /prelims-result` | **yes**: the other answer replaces it, until one of the next two rows happens ("I tapped the wrong button") |
| next stage (Summit), same `attempt` | `POST /begin-next-stage` after `cleared` | **no** (final; support can fix it, see the SME doc) |
| Prelims stage, `attempt` + 1, no answer | `POST /restart` after `not_cleared` | **no** (final) |

Nothing moves by itself: the answer only records; "Begin" and the restart move. **Begin does not
need every Prelims step** — clearing the real exam is what moves the learner (the design shows 89%
of the syllabus). The streak is never read for any decision and never written (owner ruling
2026-11-16); `bestStreak` is display-only.

### `prelims` on `/path` and `/stats`

| field | type | meaning |
|---|---|---|
| `promptDue` | boolean | show the prompt by itself now: the rules' `prelims.resultDate` is set, today (IST) is on or after it, the learner is in the Prelims stage, has no answer, and has not restarted for this result |
| `canAnswer` | boolean | the learner may open the prompt (My stats). True when an answer is pending (still changeable), or when the question is **open** and not yet used by a restart. Open = the result date is reached; or, when no result date is set, the day after `exams.exam_date`. A result date in the future keeps it closed |
| `result` | `"cleared"` \| `"not_cleared"` \| null | this run's answer (still reported after "Begin"; null again after a restart) |
| `answeredDate` | string \| null | IST day of the answer |
| `resultDate` | string \| null | the rules' result date |
| `changeable` | boolean | an answer exists and neither "Begin" nor the restart has happened |
| `nextAction` | `"begin_next_stage"` \| `"restart"` \| null | with a pending answer, the screen to resume. `POST /prelims-result` with the same answer returns it again |
| `prompt` | {`title`, `body`, `yesLabel`, `noLabel`} | the prompt copy (rules `prelims.text`) |

**"For this result".** A restart freezes the answer and its day in the ledger. The question does
not come back for the same result. It comes back when the business sets the next year's
`prelims.resultDate`. Precisely: an answer counts for the current result when it was given on or
after the result date, or after the exam day when `exams.exam_date` lies before the result date.
**The business must set the new result date every year.**

### POST /activity-path/prelims-result

Body `{"result": "cleared" | "not_cleared"}`; anything else or an extra field → `400`. Runs the
same locked pass as `/path` first, then records the answer. **Nothing else moves.**

- The same answer again: no write, the same body (a double tap gives one write and two identical bodies; proven on real Postgres).
- The other answer replaces it, with a new `answeredDate`.
- `404` no path rules for the exam.
- `409`, with the message saying which:
  - not in the Prelims stage ("Your Prelims answer is final: you are in The Summit now…");
  - not open yet ("The Prelims question opens on 2027-06-12." / "…once the exam date has passed or the result date is announced.");
  - already restarted for this result;
  - `cleared` while the rules have no stage after Prelims.

| field | type | notes |
|---|---|---|
| `result` | string | as sent |
| `answeredDate` | string | IST day |
| `changeable` | boolean | always `true` here |
| `nextAction` | string | `begin_next_stage` / `restart` |
| `cleared` | ClearedScreen \| null | set for `cleared` |
| `notCleared` | NotClearedScreen \| null | set for `not_cleared` |
| `serverDate` | string | |

**ClearedScreen** — `title`, `subtitle`, `numbersTitle` strings; `numbers` ScreenNumber[] (`days`,
`syllabus`, `best_streak`, `stars`); `phaseNumbers` PhaseNumbers; `note` string (the Monk note that
"Begin" writes, same text); `changesTitle` string; `changes` Change[] (`completed`, `unlocked`);
`footnote`; `completedStage` / `nextStage` {`key`, `label`, `number`, `camps`}; `primaryLabel`
("Begin Phase 2"); `shareLabel`. Sharing is client-side.

**NotClearedScreen** — `title`, `body`; `calibration` Calibration; `note` (the note the restart
writes); `keptTitle`; `kept` ScreenNumber[] (`camp`, `stamps`, `journal`, `stars`); `changesTitle`;
`changes` Change[]. `camp` ("Current camp position") = camps reached in this run, the same number
as `/stats` `header.campNumber`: "Camp 2 • Foothills", or "Camp 0 • On the way to Base Camp"
(`prelims.text.keptCampNoneValue`) when none is reached yet. `stamps` = every stamp ever earned =
`/stats` `stamps.length`. They may differ, under their own labels (rules v8).

**ScreenNumber** — `key`, `label`, `value` int \| null, `text` (rendered: "57%", "Camp 3 • Ridge
Trail", "—" when null). **Change** — `key`, `title`, `body`.

**Calibration** — `available` (the rules name `calibrationSimulationId` AND that simulation exists,
is active and belongs to this exam), `simulationId` \| null, `title` \| null (its name), `questions`
int \| null (`simulations.totalQuestions`), `estimatedMinutes` int \| null
(`simulations.durationMinutes` — never an invented "20 minutes"), `questionsLabel`, `timeLabel`,
`timeText` \| null ("105 min"), `link` \| null (`/open/simulation_start/<id>`: starts the timed
test), `alreadyTaken` (a SUBMITTED attempt of it exists — "done" is then accepted right away),
`startLabel`, `skipLabel`. Not available → every nullable field null and `alreadyTaken` false; the
app shows only "Skip for now".

**PhaseNumbers** (also frozen in the ledger and on `/stats` `history`) — all ints:

| field | definition |
|---|---|
| `daysOnPath` | distinct IST study days in this exam, lifetime (the active-day definition) |
| `syllabusCoveredPercent` | int \| null — the `/score` `topic_coverage` signal, rounded down; null when the learner was never active or the exam has no mapped study material |
| `bestStreak` | `user_auth.max_streak`, **display only** |
| `starsEarned` | subject stars held, lifetime |
| `stagePercent`, `stepsDone`, `stepsTotal` | the stage's progress at that moment |
| `campsCleared` | camps reached in that run under the rules of that day = `/stats` `header.campNumber` at that moment |
| `stampsEarned` | camp stamps of this stage earned in this run (a stamp stays even if a later rule edit un-clears its camp) |

**What each "What changes now" line says, and why (the mockup lines that would be false here were
changed):**

| key | shown | text (en) |
|---|---|---|
| `completed` | cleared, always | "Phase 1 (The Ascent)" / "Completed" |
| `unlocked` | cleared, always | "Phase 2 (The Summit)" / "Unlocked • 3 new camps • Answer writing begins" (camp count from the rules) |
| `streak` | not cleared, only when best streak ≥ 2 | "Your streak is not reset" / "Your best streak of 21 days is saved with your phase numbers. Starting again does not touch your streak." — the mockup's "Streak resets to 0" is false by ruling: the path never touches the streak |
| `rebuilt` | not cleared, always | "Path rebuilt around your weak areas" / "Phase 1 starts again from Base Camp. Steps that target your weakest subjects follow what the calibration test exposes (or your practice accuracy if you skip it)…" — the mockup's "next 90 days" is dropped (no such horizon exists) |
| `exam_date` | not cleared, only when `exams.exam_date` is **in the future** | "Countdown moves to the next exam" / "Your countdown now points to the next exam on May 23, 2027." — the exam date is exam-level config; the path never changes it, and no per-learner study plan exists to "recalibrate" |

The cleared footnote says "Your canvas starts fresh for Phase 2" rather than "resets to 0%":
history counts, so a learner can arrive with Phase 2 goals already met (see below).

### POST /activity-path/begin-next-stage

No body. Needs the answer `cleared` → otherwise `409 Answer "cleared" first…`.

In one transaction after the pass, it does four things:
- writes a `stage` ledger row for the completed stage, with the numbers frozen;
- writes the `stage_cleared` journal note;
- moves the progress row to the next stage, with the same `attempt` and slot fills cleared;
- runs the new stage's first pass.

From then on, `/path` serves stage 2 with its own camps. `/stats` keeps every stamp, and
`header.stagePercent` follows stage 2. "Completed The Ascent" (`stage_completed` titles) is earned,
because clearing the real exam completes the stage. Repeat or double tap: `200` with the same body
and `applied: false`, and nothing written twice (proven on real Postgres). After it, `/prelims-result`
→ `409` (final) and `/restart` → `409`.

Response: `stage` (as `/path` `stage`), `attempt`, `completed` HistoryEntry, `applied`, `serverDate`.

**Stage 2 from day one.** Summit goals count **lifetime** history ("history counts", ruled for the
whole path). Its active-day goals are windowed ("3 of the last 7 days"). `mains_evaluated` and
`topics_completed` count everything ever done. So a learner who already has Mains evaluations
or topics can meet Summit goals on arrival. Real run: Summit step 1 showed "0 of 1 done" (Mains)
plus two goals already "Completed". If the first pass of the new stage clears several camps,
only the highest celebrates (the rest are stamped seen, like a first-ever open). **Owner question
— see the B7 report.**

### POST /activity-path/restart

Body `{"calibration": "done" | "skipped"}`; anything else → `400`. Needs the answer `not_cleared` →
otherwise `409`. `done` → `409` when no calibration test is available, or when the learner has no
SUBMITTED attempt of it ("No submitted attempt of the calibration test was found. Submit it first,
or skip for now."); the app then offers "Skip for now".

In one transaction after the pass, it does five things:
- writes a `stage` row for the run that ends, with numbers, answer and calibration frozen;
- writes the `stage_restart` note;
- sets `attempt` + 1, back to the first stage and its first camp, with `prelims_result` cleared;
- records `calibration_attempt_id`;
- pre-fills the slots, then runs the new run's first pass.

**Nothing is deleted.** The ledger key includes `attempt`, so the old run's goal, step and camp rows
stay. Stamps, stars, titles and notes stay. **History counts on the new run**: a strong learner
re-clears the early steps on the next open (real run: Base Camp re-stamped as attempt 2, one
celebration).

**"Weak subjects first" (calibration = done).** Every `weakest_subject` / `second_weakest_subject`
goal of the stage is filled at the restart, in this order:
1. The subjects of the calibration attempt, ranked by correct ÷ presented. Skips count as not
   correct. Mock subjects map to path subjects through `mockSubjects`; unmapped ones are ignored.
   A subject needs ≥ 5 questions in the test to be ranked. Ties go to more questions, then rule
   order.
2. Then the goal's remaining candidates, by practice accuracy (the normal slot rule).

The fills are frozen like any fill. Not pre-filled:
- `least_covered_subject` goals: they rank by study-material coverage, which a timed test does
  not measure;
- goals with a fixed subject: they are rule content;
- everything, when calibration is `skipped` or nothing maps. The normal lazy rule then applies,
  which is the practice-accuracy fallback.

Response: `stage`, `attempt` (the new one), `calibration`, `calibrationAttemptId` \| null,
`focusSubjects` [{`key`, `label`, `accuracyPercent`, `questions`}] weakest first, `slotFills`
{goalKey: subjectKey}, `restarted` HistoryEntry, `applied`, `serverDate`. A repeat or double tap
returns `applied: false` with the same body, and the attempt goes up exactly once (proven on real
Postgres, also with an answer racing the restart). A later `/prelims-result` for the same result →
`409`.

**HistoryEntry** (`/stats` `history[]`, `completed`, `restarted`) — `kind` `"stage_completed"` \|
`"stage_restarted"`, `stageKey`, `stageLabel`, `stageNumber`, `attempt` (the run that ended), `date`,
`label` ("The Ascent completed • Nov 16, 2026"), `numbers` PhaseNumbers. All of it is read from the
frozen ledger payload.

### B7 Example 1 — cleared, end to end (real responses, throwaway Postgres, en)

Learner with 8 study days, 275 PYQ answers and Base Camp stamped (`seed-b7b`); rules override
`{"prelims": {"resultDate": "2026-11-16"}, "calibrationSimulationId": "sim-cal"}`.

`POST /activity-path/prelims-result {"result": "cleared"}`:

<!-- example: prelims-result-response | rules v8 + override {resultDate, calibrationSimulationId} | veteran answers cleared, en (v7 recording; note updated by hand for v8) -->
```json
{
  "result": "cleared",
  "answeredDate": "2026-11-16",
  "changeable": true,
  "serverDate": "2026-11-16",
  "nextAction": "begin_next_stage",
  "cleared": {
    "title": "You made it through",
    "subtitle": "Phase 1 completed. A new climb begins.",
    "numbersTitle": "Your phase 1 in numbers",
    "numbers": [
      {
        "key": "days",
        "label": "Days on the path",
        "value": 8,
        "text": "8"
      },
      {
        "key": "syllabus",
        "label": "Syllabus covered",
        "value": 57,
        "text": "57%"
      },
      {
        "key": "best_streak",
        "label": "Best streak",
        "value": 1,
        "text": "1"
      },
      {
        "key": "stars",
        "label": "Subject stars earned",
        "value": 10,
        "text": "10"
      }
    ],
    "phaseNumbers": {
      "daysOnPath": 8,
      "syllabusCoveredPercent": 57,
      "bestStreak": 1,
      "starsEarned": 10,
      "stagePercent": 16,
      "stepsDone": 5,
      "stepsTotal": 30,
      "campsCleared": 1,
      "stampsEarned": 1
    },
    "note": "57% of the syllabus covered. Prelims was just the first gate, and you walked through it. The Summit is a different kind of climb. Let's go.",
    "changesTitle": "What changes now",
    "changes": [
      {
        "key": "completed",
        "title": "Phase 1 (The Ascent)",
        "body": "Completed"
      },
      {
        "key": "unlocked",
        "title": "Phase 2 (The Summit)",
        "body": "Unlocked • 3 new camps • Answer writing begins"
      }
    ],
    "footnote": "Your canvas starts fresh for Phase 2, but all your stars, stamps and phase 1 scores are saved and always visible in your journal.",
    "completedStage": {
      "key": "ascent",
      "label": "The Ascent",
      "number": 1,
      "camps": 6
    },
    "nextStage": {
      "key": "summit",
      "label": "The Summit",
      "number": 2,
      "camps": 3
    },
    "primaryLabel": "Begin Phase 2",
    "shareLabel": "Share this moment"
  },
  "notCleared": null
}
```

`POST /activity-path/begin-next-stage`:

<!-- example: begin-next-stage-response | rules v7 + override | the same veteran taps Begin Phase 2 -->
```json
{
  "stage": {
    "key": "summit",
    "label": "The Summit",
    "examPart": "mains",
    "number": 2,
    "totalStages": 2
  },
  "attempt": 1,
  "completed": {
    "kind": "stage_completed",
    "stageKey": "ascent",
    "stageLabel": "The Ascent",
    "stageNumber": 1,
    "attempt": 1,
    "date": "2026-11-16",
    "label": "The Ascent completed • Nov 16, 2026",
    "numbers": {
      "daysOnPath": 8,
      "syllabusCoveredPercent": 57,
      "bestStreak": 1,
      "starsEarned": 10,
      "stagePercent": 16,
      "stepsDone": 5,
      "stepsTotal": 30,
      "campsCleared": 1,
      "stampsEarned": 1
    }
  },
  "applied": true,
  "serverDate": "2026-11-16"
}
```

Then `GET /path` (excerpt — every other key as in Example 1 of `/path`): `"stage": {"key":
"summit", "number": 2}`, camps `summit.c1` current / `c2`, `c3` locked, `current.stepKey`
`summit.c1.s1` with goals `mains_evaluated` "0 of 1 done", `active_days` "Completed",
`topics_completed` "Completed", `"chips": {"stagePercent": 0, "starsEarned": 10, "starsTotal": 33}`,
celebrations `the_star_collector` (still unseen from the first open) and **`the_mountaineer`**,
`"prelims": {"promptDue": false, "canAnswer": false, "result": "cleared", "answeredDate":
"2026-11-16", "changeable": false, "nextAction": null, …}`. `/stats` after: `header.stageLabel`
"The Summit", `stagePercent` 0, `nextStageLabel` null; `stamps` still `[ascent.c1 (attempt 1)]`;
`stars.earned` 10; `journal.total` 3 (was 2; `latest` = the stage note); `history` = [the
`completed` entry above]. Every ledger row present before "Begin" was still there, unchanged.

### B7 Example 2 — not cleared, calibration taken, restart (real responses, en)

`POST /activity-path/prelims-result {"result": "not_cleared"}` (learner with 17 study days, best
streak 30; the exam date on the row is 2027-05-23, in the future):

<!-- example: prelims-result-response | rules v8 + override | learner u_b answers not_cleared (calibration submitted, exam date 2027-05-23), en (v7 recording; note and camp position updated by hand for v8) -->
```json
{
  "result": "not_cleared",
  "answeredDate": "2026-11-16",
  "changeable": true,
  "serverDate": "2026-11-16",
  "nextAction": "restart",
  "cleared": null,
  "notCleared": {
    "title": "Let's see where you really stand",
    "body": "Take a simulation of this year's Prelims paper. It helps me understand where you lost marks and build your path around those areas, not just estimates.",
    "calibration": {
      "available": true,
      "simulationId": "sim-cal",
      "title": "UPSC Prelims 2026 Calibration",
      "questions": 35,
      "estimatedMinutes": 105,
      "questionsLabel": "Questions",
      "timeLabel": "Estimated time",
      "timeText": "105 min",
      "link": "/open/simulation_start/sim-cal",
      "alreadyTaken": true,
      "startLabel": "Start calibration test",
      "skipLabel": "Skip for now"
    },
    "note": "One result does not undo the work you put in. What we do now is smarter, not harder. I know exactly where to look.",
    "keptTitle": "Everything you built stays",
    "kept": [
      {
        "key": "camp",
        "label": "Current camp position",
        "value": 0,
        "text": "Camp 0 • On the way to Base Camp"
      },
      {
        "key": "stamps",
        "label": "Stamps",
        "value": 0,
        "text": "0"
      },
      {
        "key": "journal",
        "label": "Journal entries",
        "value": 2,
        "text": "2"
      },
      {
        "key": "stars",
        "label": "Subject stars earned",
        "value": 2,
        "text": "2"
      }
    ],
    "changesTitle": "What changes now",
    "changes": [
      {
        "key": "streak",
        "title": "Your streak is not reset",
        "body": "Your best streak of 30 days is saved with your phase numbers. Starting again does not touch your streak."
      },
      {
        "key": "rebuilt",
        "title": "Path rebuilt around your weak areas",
        "body": "Phase 1 starts again from Base Camp. Steps that target your weakest subjects follow what the calibration test exposes (or your practice accuracy if you skip it), not where you were already strong."
      },
      {
        "key": "exam_date",
        "title": "Countdown moves to the next exam",
        "body": "Your countdown now points to the next exam on May 23, 2027."
      }
    ]
  }
}
```

`POST /activity-path/restart {"calibration": "done"}`. The calibration attempt had:
- Economics: 12 questions, 3 right, 2 skipped;
- Polity: 8 questions, 6 right;
- Art and Culture: 10 questions, 9 right;
- Governance: 5 questions (unmapped, so ignored).

<!-- example: restart-response | rules v7 + override | u_b restarts with calibration=done -->
```json
{
  "stage": {
    "key": "ascent",
    "label": "The Ascent",
    "examPart": "prelims",
    "number": 1,
    "totalStages": 2
  },
  "attempt": 2,
  "calibration": "done",
  "calibrationAttemptId": "att-cal-b",
  "focusSubjects": [
    {
      "key": "economy",
      "label": "Economy",
      "accuracyPercent": 25,
      "questions": 12
    },
    {
      "key": "polity",
      "label": "Polity",
      "accuracyPercent": 75,
      "questions": 8
    },
    {
      "key": "art_culture",
      "label": "Art & Culture",
      "accuracyPercent": 90,
      "questions": 10
    }
  ],
  "slotFills": {
    "ascent.c1.s5.accuracy-weakest_subject": "economy",
    "ascent.c3.s2.gap-weakest_subject": "economy",
    "ascent.c4.s4.gap-second_weakest_subject": "polity",
    "ascent.c5.s1.gap-weakest_subject": "economy",
    "ascent.c6.s2.gap-second_weakest_subject": "polity",
    "ascent.c6.s4.gap-weakest_subject": "economy"
  },
  "restarted": {
    "kind": "stage_restarted",
    "stageKey": "ascent",
    "stageLabel": "The Ascent",
    "stageNumber": 1,
    "attempt": 1,
    "date": "2026-11-16",
    "label": "The Ascent restarted • Nov 16, 2026",
    "numbers": {
      "daysOnPath": 17,
      "syllabusCoveredPercent": 24,
      "bestStreak": 30,
      "starsEarned": 2,
      "stagePercent": 0,
      "stepsDone": 0,
      "stepsTotal": 30,
      "campsCleared": 0,
      "stampsEarned": 0
    }
  },
  "applied": true,
  "serverDate": "2026-11-16"
}
```

After it, `activity_path_progress` = `attempt 2`, stage `ascent`, those six fills (plus the
lazily resolved `ascent.c1.s1.topics-least_covered_subject` once step 1 was current),
`prelims_result` NULL, `calibration_attempt_id` `att-cal-b`. `GET /path` → `"attempt": 2`, current
`ascent.c1.s1`, `"prelims": {"promptDue": false, "canAnswer": false, "result": null, …}`. A repeat
`/restart` → `applied: false`, attempt still 2. `/prelims-result` → `409 You already answered for
this Prelims result and restarted the path…`.

**Skipped, strong learner (stats before → after).** The learner had Base Camp stamped, 10 stars,
"The Star Collector" and 2 notes. `restart {"calibration": "skipped"}` gave `slotFills: {}`,
`focusSubjects: []`, `calibrationAttemptId: null`. Fills then resolved lazily from practice on the
next open (`ascent.c1.s5.accuracy-weakest_subject` → `polity`). `GET /path` on attempt 2 showed:
- Base Camp cleared again and current `ascent.c2.s1`;
- one camp celebration (`ascent.c1`).

`/stats` after:
- `stamps`: `[ascent.c1 attempt 1, ascent.c1 attempt 2]`;
- `stars.earned` 10 (unchanged) and title unchanged;
- `journal.total` 3 (the `stage_restart` note is `latest`; recorded under v7 as "You showed up for
  8 days…", under v8 the same learner — 8 days, 57% — gets "57% of the syllabus is already behind
  you…");
- `history` = [`stage_restarted`, numbers `{daysOnPath 8, syllabusCoveredPercent 57, bestStreak 1,
  starsEarned 10, stagePercent 16, campsCleared 1, stampsEarned 1}`].

Ledger rows per kind before → after:
- before: `0|note 2, 0|star 10, 0|title 2, 1|camp 1, 1|goal 19, 1|step 5`;
- after: the same plus `0|note +1, 1|stage 1, 2|camp 1, 2|goal 19, 2|step 5`.

Every row present before the restart was still there, with the same payload.

### B7 — proven on real Postgres (throwaway `postgres:17-alpine`, full migration history, `en_US.utf8`)

- **IST boundary.** Result date = today. At 18:29:59.999Z the day before, the result was
  `serverDate` 2026-11-15 with `promptDue` false. At 18:30:00Z, `serverDate` was 2026-11-16 with
  `promptDue` true.
- **Defaults** (no result date; exam date in the future): `promptDue` and `canAnswer` were false,
  and an answer got `409`.
- **Answers.** The same answer twice gave an identical body and one write. cleared → not_cleared →
  cleared changed the answer each time.
- **Double taps** on the real advisory lock:
  - answer: identical bodies;
  - begin: `[applied false, true]`, one `stage` row;
  - restart: `[false, true]`, attempt 2 once;
  - restart + answer racing: the answer got `409`.
- **Ledger integrity.** No duplicate (user, exam, attempt, kind, key) anywhere.
- **Journal.** On the first page, the stage note sorted directly above that day's regular note.
  Page 2 continued with the day before.
- **Staging (read-only) timing of the new reads:**

  | read | time |
  |---|---|
  | `stage` rows | 0.13 ms (index `user_id, exam_id, kind, earned_at`) |
  | journal page with byte-order key | 0.07 ms |
  | `exams.exam_date` | 0.10 ms |
  | `max_streak` | 0.02 ms |
  | latest calibration attempt | 0.05 ms (index `userId, simulationId`) |
  | per-subject tallies | 1.7 ms |
