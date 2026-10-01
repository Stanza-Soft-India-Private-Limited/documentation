# SME — Insight Endpoints (question quality · release health · notification effectiveness · onboarding funnel · purchase funnel)

> Changed 2026-09-21 — stale-text pass (telemetry live in 2.0, version ladder, links). Exam-dimension changes follow in [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md).
> Changed 2026-09-21 — exam-safe ingest: /sme/content/* + examIds on /cms and content-doc (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).
> Changed 2026-09-21 — ?exam= on every analytics route, examCoverage, purchase-funnel (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

> ## ⛔ BEFORE YOU WRITE ANY UI CODE — INVOKE THE `frontend-design` SKILL
> If you are a Claude/coding session building screens on these endpoints, your **first
> action** is to invoke the **`frontend-design`** skill (`/frontend-design`). Do this **even
> if no one asked you to** and even if the surrounding task didn't mention design. Every
> table, chart, funnel and empty state below must be designed through that skill — do not
> reach for a default dashboard template. This is a hard requirement, not a suggestion.
>
> And design **four different screens**, because these are four different jobs. The single
> most common failure here is rendering all four as identical "analytics cards" on one
> dashboard. They are not:
>
> | Endpoint | What the screen actually IS | Who reads it | Its job |
> |---|---|---|---|
> | `question-quality` | a **work queue** — a to-do list of items to open and fix | the content team, weekly | hand someone the next broken question, with the evidence attached |
> | `release-health` | a **go/no-go decision screen** | whoever owns the release, once per release | answer "do we force-update?" with a number, and nothing else |
> | `notification-effectiveness` | a **kill list** | growth/product, monthly | name the notification to stop sending |
> | `onboarding-funnel` | a **diagnostic** — where the money leaks | founders/ops, continuously | point at the one stage worth fixing next |
> | `purchase-funnel` *(new)* | a **conversion diagnostic** — where the money *stops* | growth/founders, weekly | show which purchase step loses people, per day |
>
> A work queue wants rows, a claim/open action and a sort you can trust. A go/no-go wants
> one number the size of a headline. A kill list wants exactly one recommendation visible
> at a time. A funnel wants stage-to-stage loss, not five equal-weight tiles. If your four
> screens look the same, you have built a template, not a tool.
>
> **None of these are polling tiles.** Every one is a bounded sequential scan over the
> production database the mobile app runs on (§6). Build them as pages you *open*, with an
> explicit refresh — never as widgets on an auto-refreshing home screen.
>
> Paste this with the doc when you brief an agent:
>
> ```
> Invoke the /frontend-design skill first, before writing any UI code.
> Then build the five SME insight screens per SME_INSIGHTS_API.md.
> They are five DIFFERENT surfaces, not five cards on one dashboard:
>   question-quality        = a content-team WORK QUEUE (rows + an action per row)
>   release-health          = a release-owner GO/NO-GO decision (one headline number)
>   notification-effectiveness = a growth KILL LIST (one recommendation at a time)
>   onboarding-funnel       = an ops DIAGNOSTIC (stage-to-stage loss, biggest drop first)
>   purchase-funnel         = a CONVERSION diagnostic (per-day steps + a conversion line)
> Responses are RAW JSON — there is no {success,data} envelope; branch on HTTP status.
> Every response carries meta.caveats: string[] (purchase-funnel: top-level caveats[])
>   — surface it, do not drop it.
> Render null as "not recorded yet" / "not derivable", never as 0 and never as a green tick.
> Every /sme/analytics/* route takes an optional ?exam= — read §0.1 before wiring a filter.
> These are review reads over the production DB: open-and-refresh pages, never polling tiles.
> ```

Five read-only decision endpoints for the SME portal. Each one exists because a
decision is currently being made by guessing:

| Endpoint | The guess it replaces |
|---|---|
| `GET /sme/content/question-quality` | "which questions are bad?" — today the content team eyeballs it |
| `GET /sme/analytics/release-health` | "should we force-update?" — today it's a hunch |
| `GET /sme/analytics/notification-effectiveness` | "is this push working?" — today the only lever is *send more* |
| `GET /sme/analytics/onboarding-funnel` | "where do signups die?" — today nobody can answer at all |
| `GET /sme/analytics/purchase-funnel` | "where does the checkout lose people?" — today only the paid orders are visible |

**Base URL:** `{{BASE_URL}}/api/v1` (prod `https://app.stanzasoft.ai`)
**Auth:** `x-api-key: <API_KEY_SECRET>` on every request · **Swagger:** `/api/docs` (group **SME**)
All day-bucketing is **IST** (UTC+5:30) via the one shared helper, `common/utils/ist-date.util`.

---

## ⚠️ 0. Responses are RAW — there is no envelope

**Do not write a client that unwraps `{ success, data }`.** `ResponseInterceptor`
exists in this codebase but is **never registered**; the sole global
`APP_INTERCEPTOR` is `ApiUsageInterceptor`. Success bodies go out exactly as the
DTO interfaces declare them:

```jsonc
// GET /sme/analytics/release-health  →  200
{ "versions": [ … ], "platforms": [ … ], "meta": { … } }   // ✅ this
// NOT { "success": true, "data": { … } }                  // ❌ never this
```

Errors are the Nest default `{ statusCode, message }` from the global exception
filter. (The one structured pass-through anywhere in this API is the 402 quota
body; nothing here returns 402.)

Every response carries a `meta` block. **Read it.** It states the window actually
used, the thresholds actually applied, whether a row cap bit, and the caveats that
make the numbers honest. `meta.caveats` is a ready-to-render `string[]` — put it
behind an "i" on the panel rather than reprinting it in a deck. (`purchase-funnel`,
§5, is the one exception to the `meta` convention: it is a small enough response that
`exam`, `days` and `caveats` sit at the top level and there is no `meta` block.)

## ⚠️ 0.1 `?exam=` — additive on every analytics route here

Every `/sme/analytics/*` route — §2, §3, §4 and the new §5 — accepts an optional
**`?exam=<slug>`** as of **2026-09-21**. `GET /sme/content/question-quality` (§1) does
**not**; that gap is still open and is listed in §7.

**Omit it and nothing changes.** The response is byte-identical to what the route has
always returned: no new keys, no wrapper. **An unknown slug is a `400`** on every one
of them — never a silently empty chart, because on a decision screen "0" and "you
typo'd the slug" look identical and only one is worth acting on.

Supply it and two rules govern how honest the number can be. They **point in opposite
directions**, which is the whole trap:

* **Person-keyed** (`user_profiles.active_exam_id`) — §3, §4's cohort, and §5's event
  steps. **NULL ≡ `upsc-cse`**: the person onboarded before app 2.0 (2026-09-17, which
  made the exam picker onboarding step 1) and has never used the switcher, so the
  default is what they were served. Full history — but **retro-attributed**: a user is
  counted under the exam they have picked *today*, applied backwards over everything
  they ever did. There is no switcher history.
* **Request-keyed** (`api_usage.exam_id`) — §2. **NULL = "not recorded" and the row is
  EXCLUDED**, `upsc-cse` included, because the column only exists from the 2026-09-21
  deploy. Those responses carry **`examCoverage { since, unknownRows }`** so the portal
  can draw a start date instead of a cliff.

And two places where a per-exam number **cannot** be honest, said out loud rather than
quietly fudged — both because `auth_events` has no exam column and no relation to join
one through:

* §2's `clientErrors` stays **app-wide** beside exam-scoped actives
  (`meta.clientErrorsExamScoped: false`);
* §4's whole `authEvents` block stays **app-wide** beside an exam-scoped funnel.

The per-route reference table lives in
[SME_USAGE_ANALYTICS.md §0b](./SME_USAGE_ANALYTICS.md) — that doc is the field-level
contract; this one is the narrative.

---

## 1. `GET /sme/content/question-quality`

> An item with an anomalous wrong-answer rate is usually a **bad question** or a
> **bad explanation**, not a hard one. This is the work queue for finding them.

### Why it is not sorted by wrong-rate

Sorting by raw wrong-rate returns whatever was answered twice and missed twice.
So the ranking is:

```
optionCount         = how many options the question actually has (C/D are nullable
                      on content_questions, so a 2-option item is scored as 2-option)
chanceWrongRate     = 1 − 1/optionCount            → 0.75 for a normal 4-option MCQ
confidentWrongRate  = Wilson 95% LOWER bound on wrong/attempts
priorityScore       = max(0, confidentWrongRate − chanceWrongRate) × attempts
```

`priorityScore` reads as **"learners we are 95% confident this item misled, beyond
what guessing alone would have cost them."**

The anchor is not arbitrary: **a well-formed MCQ cannot perform worse than random
guessing.** If it does, the key is wrong, two options are both defensible, or the
stem is ambiguous — a defect, not difficulty. The Wilson lower bound (rather than
the raw rate) is what kills small samples: it collapses toward 0 as *n* shrinks and
converges on the raw rate as *n* grows, so a 90%-wrong item seen by 500 people
outranks a 100%-wrong item seen by 12, and a 3-attempt item scores exactly 0.

| `flag` | Meaning | What to do |
|---|---|---|
| `worse_than_chance` | Wilson bound clears the guessing rate — the item is measurably harming people | Fix or pull it. Check the **key** first. |
| `suspicious` | Wilson bound clears `suspectRate` (default 60%) but not the guessing rate | Probably just hard. Check the explanation. |
| `ok` | Neither | Leave it alone. |

**`minAttempts` (default 20, min 5, max 1000).** Not a round number for its own
sake: with 4 options the Wilson bound only clears 75% from *n ≈ 12* upward *even
when every single answer is wrong*, so 20 leaves headroom above the point where the
flag can fire at all. Lower it to 10 for brand-new content — the statistics stay
honest, the list just gets shorter.

### Query

| Param | Default | Notes |
|---|---|---|
| `pool` | `both` | `content` \| `simulation` \| `both`. Every row carries its own `pool`. |
| `exam` | — | Exam mode slug, e.g. `appsc-group-1`. Matches that exam **or** the `*` (every-exam) sentinel. Omit to score every exam together. **Unknown slug = 400**, never a silently empty work queue. Max 64 chars. |
| `subject`, `topic` | — | Exact match. content pool → `content_documents`; simulation pool → `simulation_questions`. |
| `minAttempts` | `20` | 5–1000. Pushed into SQL as a `HAVING`. |
| `suspectRate` | `0.6` | 0.3–0.99. Only affects the `suspicious` band; `worse_than_chance` uses the per-question guessing rate. |
| `flag` | — | Severity floor: `worse_than_chance` \| `suspicious` \| `ok`. |
| `sort` | `priority` | `priority` \| `wrongRate` \| `attempts`. The last two are for cross-checking, not for triage. |
| `includeInactive` | `false` | `true` to include already-deactivated questions. |
| `days` / `from` / `to` | `90` | Max 365. `YYYY-MM-DD` is read as an **IST** calendar date. |
| `limit` | `50` | Max 200. |

### Response (abridged)

```jsonc
{
  "items": [{
    "pool": "content",
    "questionId": "…",
    "questionText": "Which of the following …",   // truncated at 300 chars
    "questionTextTruncated": true,
    "subject": "Polity", "topic": "Constitution",
    "documentId": "…", "documentTitle": "Fundamental Rights",
    "position": "3.7",                            // section.question, content pool only
    "simulationNames": [],                        // simulation pool only
    "correctAnswer": "C", "optionCount": 4, "isActive": true,
    "hasExplanation": false, "explanationLength": 0,

    "attempts": 240, "wrong": 199, "wrongRatePct": 82.9, "skipped": 0,
    "chanceWrongRatePct": 75, "confidentWrongRatePct": 77.7,
    "flag": "worse_than_chance", "priorityScore": 6.5,

    "optionDistribution": [
      { "option": "B", "count": 150, "sharePct": 62.5, "isKey": false },
      { "option": "C", "count": 41,  "sharePct": 17.1, "isKey": true  }
    ],
    "topWrongOption": "B", "topWrongOptionSharePct": 62.5,
    "contradictsKey": true,
    "reason": "82.9% wrong over 240 answers — worse than the 75% you would get by guessing… 62.5% picked B while the key says C — check the key first. No explanation is attached, so nobody who missed it learned why."
  }],
  "summary": { "eligible": 812, "worseThanChance": 9, "suspicious": 41, "ok": 762, "totalLearnersMisled": 143.2 },
  "meta": { "pools": ["content","simulation"], "exam": null, "minAttempts": 20, "candidateCapHit": false, "caveats": [ … ] }
}
```

`meta.exam` echoes the **resolved** slug — trimmed and lower-cased — or `null` when no
`?exam=` was sent and the queue spans every exam. Render it: a work queue that silently
covers one exam and a work queue that covers all of them look identical otherwise.

`contradictsKey: true` (the most-picked option is **not** the key and beats it) is
the single highest-yield signal in the payload — it almost always means the answer
key is wrong.

### What this data can and cannot tell you

* **`user_question_attempts` is UPSERTED on `(user_id, question_id)`** — one row per
  user per question. So `attempts` means *distinct users who answered*, and
  `isCorrect` is that user's **latest** answer. Someone who missed it, retried and
  got it right counts as **correct**. The wrong-rate here is therefore a **floor**
  on first-attempt wrongness, never an overstatement.
* The content-pool window is applied to **`updated_at`**, not `created_at` —
  because the row is upserted, `updated_at` is when the correctness you are reading
  was actually decided.
* **The simulation pool has no per-answer `isCorrect`**; correctness is derived
  against the question's *current* `correctAnswer`. If a key was corrected after the
  fact, historic answers are re-scored under the new key.
* `simulation_answers` has **no timestamp of its own**, so the window is applied to
  the parent `simulation_attempts.startedAt` and only **submitted** attempts count.
  Blank answers are reported as `skipped` and excluded from `attempts` — a skip is
  not a wrong answer.
* **`?exam=` scopes the queue, and the tag lives on the PARENT, never on the question.**
  The content pool filters on `content_documents.examIds` (on the row directly); the
  simulation pool filters on the `examIds` of the simulations each question is linked into
  via `SimulationQuestionLink` — `simulation_questions` itself carries no `examIds`. The slug
  is resolved **before any query runs**, so an unknown one is a 400 rather than an empty work
  queue, and `meta.exam` echoes what was applied.

  ⚠️ **A simulation question assigned to no test is unreachable under an exam filter.** That
  is by design: it is published nowhere, so it belongs to no exam's catalogue. It is still
  scored in the unfiltered call.

  The `meta.caveats` entry says which case you got — one of these two strings, verbatim:

  > `exam filter APPLIED (appsc-group-1, plus the "*" every-exam sentinel). The tag lives on the parent, not the question: the content pool is filtered on content_documents.examIds, the simulation pool on the examIds of the simulations each question is linked into — so a simulation question assigned to no test cannot appear under an exam filter.`

  > `No exam filter: every exam is scored together. Pass ?exam=<slug> to scope the queue — the content pool filters on content_documents.examIds and the simulation pool on the parent simulations' examIds.`
* Neither attempts table has a **timestamp index**, so this is a bounded sequential
  scan of the window. It is a review read, not a dashboard tile — do not poll it.

### How to use this data

**Questions it answers.** "Which question should the content team open next?" · "Is this
item hard, or is the key wrong?" · "Did anyone learn anything from getting it wrong?"
(`hasExplanation` / `explanationLength`).

**The decision it drives.** Fix the key, rewrite the explanation, or leave it alone. That
is the entire decision space, and the payload is shaped to make it in one glance:
`contradictsKey` → check the key; `flag = worse_than_chance` → pull the item;
`hasExplanation = false` on a high wrong-rate → write the explanation.

**Sort and filter on `contradictsKey` first — it is the highest-yield field in the
payload.** `priorityScore` is the statistically correct ranking and it is the right default
*at scale*, but at today's volume it cannot separate anything (see the live numbers below).
`contradictsKey` needs no sample size to be interesting: it says the most-picked option beat
the marked key, which is the signature of a **wrong answer key**, not a hard question.
Make it a first-class filter chip, not a column at position eleven.

**What a good visualisation is.** A table you can work through, not a chart. One row per
question: truncated stem, subject/topic, `attempts`, `wrongRatePct` shown *against*
`chanceWrongRatePct` (a bar with the guessing rate as a marked threshold reads instantly —
"above the line = defect"), the flag as a chip, and a `contradictsKey` badge that is
impossible to miss. Expanding a row shows `optionDistribution` as a small horizontal bar
set with the key marked — that single visual is what convinces a reviewer the key is wrong,
because they see 41.7% on C and 16.7% on the option marked correct. Use the server's
`reason` string as the row's explanation; it is written to be read verbatim.

**The action that follows.** Open the question in the CMS, fix the key or the explanation,
deactivate if unsalvageable. The queue should let a reviewer mark a row as handled — the API
has no such state, so hold it in the portal.

#### What it actually returns today — measured 2026-07-23, `days=90&minAttempts=20`

```
6 items eligible · 0 worse_than_chance · 1 suspicious · 5 ok · totalLearnersMisled 0
```

Read that honestly: at the default `minAttempts=20`, **the entire content bank produced six
eligible questions in ninety days.** Not six bad ones — six with enough distinct answerers to
be scoreable at all. `priorityScore` was `0` on all six, because `confidentWrongRatePct`
(the Wilson lower bound) never cleared the 75% guessing rate at n≈25. So the default sort
ranks nothing, and a UI that leads with `priorityScore` will present an empty-feeling screen
that is in fact working exactly as designed.

The one flagged item is the whole argument for this endpoint:

| Field | Value |
|---|---|
| `questionId` | `00f30b23-ae32-481d-b598-08cdcc541fdd` |
| subject / topic | Economy / Economy (Prelims) |
| `attempts` / `wrong` | 24 / 20 — **83.3% wrong** |
| `chanceWrongRatePct` | 75 |
| `confidentWrongRatePct` | 64.1 (Wilson lower bound at n=24) |
| `flag` | `suspicious` |
| `correctAnswer` | **D**, picked by 4 of 24 (**16.7%**) |
| `topWrongOption` | **C**, picked by 10 of 24 (**41.7%**) |
| `contradictsKey` | **`true`** |
| `hasExplanation` | `true` |

Two and a half times as many learners picked C as picked the marked key. The flag says only
`suspicious` and the priority score says `0` — both correct, both statistically honest, both
useless for triage at n=24. **`contradictsKey: true` is the field that finds this item**, and
it is why it must be the default sort/filter for the content team until the bank has an order
of magnitude more attempts.

**Practical consequence: lower `minAttempts` for real use.** The same window at
`minAttempts=5` returns **149 eligible items, 3 suspicious, and 13 with
`contradictsKey: true`** — a queue a human can actually work through this week. The
statistics stay honest at 5 (the Wilson bound simply collapses toward 0 and nothing gets
flagged `worse_than_chance` on thin evidence); you just stop pretending the flag is the only
signal. Ship the screen with a `minAttempts` control defaulted low, and a filter chip for
`contradictsKey`.

**Expect this to stay thin for a while, and say so on screen.** `attempts` counts *distinct
users who answered*, and only content-doc MCQs and simulations — PYQ attempts live in Neo4j
and are not here at all. An empty queue means "not enough answers yet", never "the content is
clean". Write that sentence into the empty state.

---

## 2. `GET /sme/analytics/release-health`

> `AppConfig.minAppVersion` already provides the force-update **mechanism**. This
> supplies the **number**.

Per **app version × platform**: active users, requests, 4xx/5xx error rate,
`client_error` count per active user, and adoption share — plus an explicit
comparison against the next-older build on the same platform, so *"is 2.0 worse
than 1.9?"* is one glance.

### ⚠️ `app_version IS NULL` is the pre-1.7 cohort, not missing data

**Current store build is 2.0** (Android versionCode 27, iOS 2.0 build 3), released
**2026-09-17**, so the ladder the version rows walk is now 1.6 → 1.7 → 1.8 → 1.9 → 2.0. The
`app_version IS NULL` rule below is **unchanged** by that and stays exactly as written. Note
also that `AppConfig.minAppVersion` on production is **1.0.0** — i.e. **no forced update is in
effect**; this endpoint supplies the number you would use to decide on one, nothing more.

`x-app-version` shipped **with 1.7**. Every request from an older build arrives
with no version header at all. So the NULL bucket **is** "pre-1.7" — precisely the
cohort a force-update decision is about. It is reported as its own row, labelled
`"pre-1.7 (no x-app-version header)"`, sorted last (oldest), and **never** dropped,
coerced to a string, or merged into a neighbour. `platform` is NULL on the same
builds and is likewise kept separate rather than folded into ios/android.

`platforms[].preHeaderSharePct` is the headline force-update number: *what share of
this platform's active users are still on a build that predates the header*.

### ⚠️ Hard 30-day ceiling

`app_version`, `platform` and `status` live **only on the raw `api_usage` table** —
the `api_usage_daily` rollup is `day × user × route` and carries none of them — and
raw rows are purged at 30 days by the nightly job. So `days` maxes at **30**, and
`meta.windowExceedsApiUsageRetention` fires when a caller aims `from`/`to` at a
stretch that has already been purged, instead of returning a confident empty answer.

### Query

| Param | Default | Notes |
|---|---|---|
| `days` / `from` / `to` | `14` | **Max 30** — the api_usage raw retention horizon. |
| `platform` | — | `ios` \| `android` \| `web`. Omit for all, including the NULL-platform rows. |
| `exam` | — | Exam slug. **400** on an unknown one. Filters the `api_usage` half on `exam_id` (rows captured before 2026-09-21 are excluded, `upsc-cse` included) and adds `examCoverage` + two `meta` keys + two caveats. ⚠️ **`clientErrors` is NOT scoped** — see below. |

### ⚠️ With `?exam=`, one column stays app-wide — and the response says which

This is the honest half of exam-scoping this endpoint. `activeUsers`, `requests`, the
4xx/5xx counts and every rate derived from them come from `api_usage`, which has an
`exam_id`. **`clientErrors` comes from `auth_events`, which has neither an exam column
nor a relation to join one through.** There is no query that would scope it.

So rather than silently dropping the column, or silently pretending it is scoped, the
response keeps it **app-wide** beside exam-narrowed actives and flags it:

* **`meta.clientErrorsExamScoped: false`** — present only when an exam filter is on.
  Branch on it.
* a caveat, verbatim: *"clientErrors is NOT exam-scoped … with `exam` set the
  client-error column stays APP-WIDE while activeUsers/requests beside it are narrowed
  — which makes clientErrorsPerActiveUser an overstatement, not a comparable per-exam
  rate. Judge builds on errorRateExcludingPaywallPct when filtering by exam."*

**`clientErrorsPerActiveUser` is therefore inflated under an exam filter**, roughly by
the inverse of that exam's share of traffic. Every build is inflated by the same rough
factor, so the *gap* between two builds is scaled up too — which means a
`clientErrorPerUserDelta` that sat comfortably inside the **±0.5** threshold can cross
it, and `comparison.verdict` can read `worse` on that axis alone. With an exam
selected, judge on `errorRateExcludingPaywallPct` and treat the client-error axis as
unusable; compare crash rates with the exam filter **off**.

### Response (abridged)

```jsonc
{
  "versions": [{
    "appVersion": "1.7.0", "appVersionLabel": "1.7.0",
    "platform": "ios",     "platformLabel": "ios",
    "activeUsers": 412, "requests": 98430,
    "errors4xx": 3120, "errors5xx": 214,
    "errorRatePct": 3.39,                 // includes 402
    "serverErrorRatePct": 0.22,
    "paywall402": 2011,
    "errorRateExcludingPaywallPct": 1.37, // ← judge the build on THIS
    "clientErrors": 88, "clientErrorsPerActiveUser": 0.21,
    "adoptionSharePct": 71.4,
    "comparison": {
      "comparedTo": "1.6.0", "comparedToLabel": "1.6.0",
      "errorRateDeltaPoints": 0.94, "clientErrorPerUserDelta": 0.03,
      "verdict": "similar",
      "note": "Within noise of 1.6.0 (error rate 0.94 points, client errors 0.03 per active user; thresholds are 1 and 0.5)."
    }
  }],
  "platforms": [{
    "platform": "ios", "platformLabel": "ios",
    "activeUsers": 577, "requests": 141002, "versions": 3,
    "worstVersion": null, "worstVersionLabel": null, "worstVersionErrorRatePct": null,
    "preHeaderSharePct": 8.3
  }],
  "meta": { "windowDays": 14, "apiUsageRawRetentionDays": 30, "windowExceedsApiUsageRetention": false, "caveats": [ … ] }
}
```

With `?exam=` the same body gains exactly three things — `examCoverage` as a **sibling
of `versions`/`platforms`** (not inside `meta`), and two `meta` keys:

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/analytics/release-health?days=14&exam=appsc-group-1` — verbatim, with
`versions[]` and `platforms[]` trimmed to their single returned row each and the five
non-exam `meta.caveats` strings replaced by `"…"` (the two exam caveats are shown in
full, because they are the ones this section is about):

```json
{
  "versions": [
    {
      "appVersion": "1.9",
      "appVersionLabel": "1.9",
      "platform": "android",
      "platformLabel": "android",
      "activeUsers": 0,
      "requests": 0,
      "errors4xx": 0,
      "errors5xx": 0,
      "errorRatePct": 0,
      "serverErrorRatePct": 0,
      "paywall402": 0,
      "errorRateExcludingPaywallPct": 0,
      "clientErrors": 11,
      "clientErrorsPerActiveUser": 0,
      "adoptionSharePct": 0,
      "comparison": {
        "comparedTo": null,
        "comparedToLabel": null,
        "errorRateDeltaPoints": 0,
        "clientErrorPerUserDelta": 0,
        "verdict": "insufficient_data",
        "note": "No older build on this platform inside the window to compare against."
      }
    }
  ],
  "platforms": [
    {
      "platform": "android",
      "platformLabel": "android",
      "activeUsers": 0,
      "requests": 0,
      "versions": 1,
      "worstVersion": null,
      "worstVersionLabel": null,
      "worstVersionErrorRatePct": null,
      "preHeaderSharePct": 0
    }
  ],
  "examCoverage": {
    "since": "2026-09-21T14:30:34.917Z",
    "unknownRows": 7473
  },
  "meta": {
    "from": "2026-09-07T14:46:38.019Z",
    "to": "2026-09-21T14:46:38.019Z",
    "windowDays": 14,
    "platformFilter": null,
    "examFilter": "appsc-group-1",
    "clientErrorsExamScoped": false,
    "minRequestsForVerdict": 200,
    "minUsersForVerdict": 5,
    "errorRateDeltaPoints": 1,
    "clientErrorDeltaPerUser": 0.5,
    "apiUsageRawRetentionDays": 30,
    "windowExceedsApiUsageRetention": false,
    "activeUserRowCap": 20000,
    "activeUserRowCapHit": false,
    "statusRowCapHit": false,
    "clientErrorRowCapHit": false,
    "caveats": [
      "…",
      "Exam filter \"appsc-group-1\": api_usage.exam_id only exists from the 2026-09-21 deploy, and rows without it are EXCLUDED (a request captured before the column belongs to no exam — not to upsc-cse). 7473 row(s) in this window were dropped for that reason, and the exam split is only real from 2026-09-21T14:30:34.917Z onward.",
      "clientErrors is NOT exam-scoped. It comes from auth_events, which has no exam column and no relation to join one through, so with `exam` set the client-error column stays APP-WIDE while activeUsers/requests beside it are narrowed — which makes clientErrorsPerActiveUser an overstatement, not a comparable per-exam rate. Judge builds on errorRateExcludingPaywallPct when filtering by exam."
    ]
  }
}
```

> 🔴 **That capture is the failure mode this section warns about, caught live.** Look at
> the one `versions[]` row: `activeUsers: 0` and `requests: 0`, but
> `clientErrors: 11`. Both are correct and they disagree on purpose —
> `activeUsers`/`requests` come from `api_usage` and **were narrowed to APPSC**, while
> `clientErrors` comes from `auth_events` and **was not** (`clientErrorsExamScoped:
> false`). `unknownRows: 7473` is the rest of the window's traffic, discarded for
> predating the column.
>
> A panel that renders this naively says *"1.9 has 11 client errors and no users."*
> **Render `examCoverage.since` and the `clientErrorsExamScoped` caveat next to the
> numbers, or do not offer the exam filter on this screen at all.**
>
> Note `since` here is `2026-09-21T14:30:34.917Z` — minutes before the capture, because
> that is when this build was deployed to staging. On production it will be the
> production deploy instant.

`examCoverage.since` is **the start date of the per-exam view**, not of the product —
render it as *"per-exam data from 21 Sep 2026"*. When it is `null` the caveat says so
in words (*"NOT ONE row in this window carries an exam, so every number here is zero
for capture reasons rather than product reasons"*) and the panel must render as **not
recorded yet**, never as a healthy build with no errors. `unknownRows` is a discard
count for this exact window, not a backlog; it shrinks to zero as pre-deploy rows age
out of the 30-day raw retention.

### Reading the verdict

`verdict ∈ better | worse | similar | insufficient_data`, computed against the
**next-older build on the same platform** on two axes:

* `errorRateExcludingPaywallPct`, threshold **±1.0 percentage point**
* `clientErrorsPerActiveUser`, threshold **±0.5 per user**

A verdict is only rendered once **both** builds clear **≥200 requests and ≥5 active
users**; below that it says `insufficient_data` rather than a confident lie. If one
axis moves up and the other down, it says `similar` and tells you so, rather than
picking a winner.

### Caveats

* `errorRatePct` **includes 402**, which is the paywall — a product behaviour, not
  a fault. Judge a build on `errorRateExcludingPaywallPct`. A monetisation push must
  never read as a broken build.
* A user active on **two builds** inside the window counts toward **both** builds'
  `activeUsers`, so per-build adoption shares can sum above 100%. `platforms[].activeUsers`
  is the true distinct count and is the denominator for `adoptionSharePct`.
* `api_usage` only captures **authenticated** requests (the interceptor skips
  public / `x-api-key` / webhook routes), so a build that cannot get past login is
  invisible in the request columns — its signal shows up in `clientErrors`, which
  comes from `auth_events` (`event_type='client_error'`) and can therefore create a
  row for a version that made no captured request at all. That is deliberate: a
  build that only produces crashes is exactly what we are hunting.
* With `?exam=`: the api_usage half is exam-scoped from **2026-09-21 only**
  (`examCoverage`), and **`clientErrors` is not exam-scoped at all**
  (`meta.clientErrorsExamScoped: false`). A force-update decision is about a *build*,
  not an exam — so make the un-filtered view the default one on this screen and treat
  the exam filter as a drill-down.

### How to use this data

**Questions it answers.** "Is the new build worse than the old one?" · "How many people are
still on a build we no longer support?" · "If we set `minAppVersion` today, how many users do
we lock out of the app until they update?"

**The decision it drives.** Exactly one: **do we raise `AppConfig.minAppVersion`, and when?**
`AppConfig` already provides the mechanism; this endpoint provides the number that makes the
call defensible. A secondary decision — "roll back?" — comes off `comparison.verdict`.

**What a good visualisation is.** Lead with a single headline number: *the share of active
users on a build that predates the version header*, rendered as one large percentage with the
absolute user count under it. That is the go/no-go. Underneath, one row per
`version × platform` with `errorRateExcludingPaywallPct` as the primary metric and the
`comparison.verdict` chip beside it — `better` / `worse` / `similar` / `insufficient_data`,
with `comparison.note` as the tooltip. Do not plot a time series; the endpoint does not return
one and a build's error rate over a 14-day window is a scalar, not a trend. Do not show
`errorRatePct` as the headline anywhere — it includes 402, and a good monetisation week would
read as a broken release.

**The action that follows.** Set `minAppVersion`, or wait. If you set it, the pre-header user
count is the number of people who will see a blocking update screen on next launch —
communicate it before you ship, not after.

#### What it returned on 2026-07-23, `days=30` — a historical snapshot, not today's numbers

> ⚠️ Dated 2026-07-23, months before 2.0. Four store releases have landed since (1.8, 1.9, 2.0),
> so the real table now has more version rows and a much smaller NULL bucket. Re-run the
> endpoint before quoting any figure below; the *reasoning* underneath it is what still applies.

| appVersion | platform | activeUsers | requests | errorRatePct | excl. paywall | clientErrors/user | verdict |
|---|---|---|---|---|---|---|---|
| `1.7` | `ios` | **2** | 88 | 0 | 0 | 0.5 | `insufficient_data` |
| `null` → *"pre-1.7 (no x-app-version header)"* | `null` → *"unknown (no x-platform header)"* | **2,097** | 170,219 | 0.5 | 0.5 | 0 | `insufficient_data` |

**2,097 of 2,099 active users — 99.9% — are on a build that predates the version header.**
That is the force-update number, and it is the entire content of the screen today. The 1.7/iOS
row is two internal devices; `insufficient_data` on both rows is correct, not a bug — the
verdict needs ≥200 requests *and* ≥5 active users on **both** builds being compared, and
there is no older build on either platform inside the window to compare against.

The pre-1.7 cohort's error rate is **0.5%** (273 4xx + 649 5xx over 170,219 requests, of which
only 16 were paywall 402s). So the old build is not on fire — the force-update case here is
about shipping the header, telemetry and the fixes, not about an emergency.

⚠️ **Do not read `preHeaderSharePct` off a single platform row and call it the answer.** It is
computed *within* a platform rollup, and pre-1.7 builds send **no `x-platform` header either**
— so they all land in the `platform: null` rollup, where `preHeaderSharePct` is trivially
`100`, while the `ios` rollup reads `0`. Neither number is the thing you want. The headline
figure is cross-platform: sum `activeUsers` on rows where `appVersion === null`, divide by the
sum of `platforms[].activeUsers`. Here: 2,097 ÷ 2,099 = 99.9%. Compute it in the client and
label it plainly.

⚠️ **`meta.windowExceedsApiUsageRetention` was `true` at `days=30`** in that run, with the
caveat *"the part of this window before 2026-06-23T14:44:09Z is GONE"*. That is expected and
near-permanent: `days` maxes at 30 and raw `api_usage` is purged at 30 days, so the window
edge and the retention edge coincide and the flag fires on essentially every full-width
request. Render it as an informational footnote ("the oldest hours of this window may be
partly purged"), **not** as an error banner — otherwise the screen cries wolf every time
someone uses the maximum window. Use `days=14` (the default) for a clean read.

---

## 3. `GET /sme/analytics/notification-effectiveness`

> `notification_history.isRead` has existed for a long time and **nothing read it**,
> so the only lever anyone had was "send more". This is the kill-list.

### ⚠️⚠️ What `isRead` actually means — read before quoting a read-rate

Verified against `notifications.service.ts`. `isRead` is written in **exactly one
place**:

```ts
markFeedSeen(userId)          // POST /notifications/feed/seen
  prisma.notificationHistory.updateMany({
    where: { userId, isRead: false }, data: { isRead: true },
  })
```

That is a **bulk** flag. It flips **every** unread row for that user the moment the
user opens the in-app notification feed (the bell). Therefore:

* `isRead = true` means **"the recipient opened the in-app feed at some point after
  this row was created"**. It does **not** mean the user tapped this push, saw this
  push, or that the push was even delivered — **push delivery is not tracked at all**.
* Nothing distinguishes "opened *because of* this notification" from "opened for an
  unrelated reason and swept 12 old rows read along the way".
* Read-rate is therefore a property of the **recipient's habits** at least as much as
  of the notification's content. Comparing two types is only fair when they go to
  comparable audiences.
* It is still trustworthy **in one direction**: a type whose recipients almost never
  come back to the feed is not working, whatever the cause. **Use this to kill
  notifications, not to declare winners.**
* **Topic broadcasts are out of scope.** `topic_notifications` has no `isRead` at
  all — its read state is derived from `user_auth.notificationsSeenAt` at read time
  and is never persisted. This endpoint covers **personal sends only**.

`meta.isReadMeaning` restates all of this verbatim in every response, so the number
cannot travel without its caveat.

### ⚠️ Time-to-read is NOT derivable

`notification_history.read_at` **now exists** (commit `c146d94`) and
`notifications.service.ts` stamps it inside the same `markFeedSeen` `updateMany` as
`isRead`. That is still not enough: it is a **bulk feed-open** timestamp, not a
per-notification open, and every row read before the column shipped has `read_at`
NULL. The instant *this* notification was read is still not stored. So the response
returns:

```jsonc
"timeToRead": {
  "derivable": false,
  "reason": "Not derivable YET. notification_history.read_at now exists and notifications.service.ts stamps it inside the same markFeedSeen updateMany as isRead — but that is still a BULK feed-open timestamp, not a per-notification open, and rows read before the column shipped have read_at NULL. …"
}
```

Do **not** synthesise this client-side. Once there is enough post-migration history,
`read_at − sent_at` becomes quotable as *"how long until this recipient came back to
the app"* — never as *"how long until they opened this push"*. A true
per-notification metric still needs an open event, which is not part of this
endpoint.

### Maturation — why the rates aren't what you'd naively compute

A notification sent 20 minutes ago has not had a chance to be read. Leaving it in
the denominator drags every recent day toward 0% and makes any campaign look like a
failure. So rows younger than `maturationDays` (**default 3**) are excluded from
every rate and reported separately as `pending`. **All rates are quoted on the
matured sample only** (`maturedSent` / `maturedRead` / `maturedReadRatePct`). A type
with no matured sample returns `maturedReadRatePct: null` — *not* `0`.

### Query

| Param | Default | Notes |
|---|---|---|
| `days` / `from` / `to` | `30` | Max 120. `YYYY-MM-DD` is an **IST** calendar date. |
| `type` | — | Restrict to one notification `type`. |
| `maturationDays` | `3` | 0–30. Rows younger than this are `pending` and excluded from rates. |
| `minSent` | `50` | Minimum **matured** sends before a type can be named a worst performer. |
| `poorReadRatePct` | `25` | Matured read-rate % at or below which a type is a worst performer. |
| `exam` | — | Exam slug. **400** on an unknown one. Matches the **RECIPIENT's** `active_exam_id` (NULL ≡ `upsc-cse`) — see below. Adds `meta.examFilter` and one caveat; **no `examCoverage`**, because nothing here is `api_usage`-sourced. |

### ⚠️ `?exam=` means "who received it", not "what it was about"

`notification_history` has **no exam column**, and the rule engine that sends these does
not record which exam a send was reasoned about. So the filter matches through the
recipient's `user_profiles.active_exam_id`, **read as of now**. Two consequences:

* it reads as *"pushes that **landed on people who study** X"*, never *"pushes **about**
  X"* — a `streak_reminder` to an APPSC aspirant is an APPSC row here whatever the
  content was;
* a user who switched exam **takes their whole send history with them**, so yesterday's
  switch re-labels months of sends.

The response ships that sentence as a caveat with the slug interpolated, and
`meta.examFilter` echoes the resolved slug. Everything else — the maturation split, the
`worstPerformers` kill-list, `timeToRead` — is unchanged in shape and meaning.

⚠️ **Do not rank kill-list candidates per exam.** `worstPerformers` is ordered by
`wastedSends`, and narrowing the audience shrinks every volume on the screen — a type
worth killing globally can fall below `minSent` (default 50) inside one exam and simply
vanish from the list. Kill on the un-filtered view; use the exam filter to check whether
a decision lands differently for one audience.

### Response (abridged)

```jsonc
{
  "byType": [{                       // sorted WORST first — this is a kill list
    "type": "streak_reminder",
    "sent": 4120, "read": 610,
    "maturedSent": 3980, "maturedRead": 597, "maturedReadRatePct": 15,
    "pending": 140, "wastedSends": 3383, "recipients": 1204,
    "firstSentAt": "…", "lastSentAt": "…"
  }],
  "byDay": [{ "day": "2026-07-23", "sent": 210, "read": 31, "readRatePct": 14.8 }],
  "worstPerformers": [{
    "type": "streak_reminder", "maturedSent": 3980, "maturedReadRatePct": 15, "wastedSends": 3383,
    "recommendation": "3383 of 3980 sends went unseen (15% read). Kill or heavily re-target 'streak_reminder' — the volume is doing more to train people to ignore the bell than to bring them back."
  }],
  "totals": { "sent": …, "read": …, "maturedSent": …, "maturedRead": …, "maturedReadRatePct": …, "pending": …, "wastedSends": …, "types": 7 },
  "timeToRead": { "derivable": false, "reason": "…" },
  "meta": { "maturationDays": 3, "maturedBefore": "…", "isReadMeaning": "…", "dailyRowCapHit": false, "caveats": [ … ] }
}
```

`worstPerformers` is ordered by **`wastedSends`**, not by read-rate: the type that
wasted the most sends is the one worth killing first. A 0%-read type sent 60 times
matters less than a 15%-read type sent 4000 times.

### Caveats

* `byType.sent` / `read` are **exact** (a DB-side aggregate). The per-day series,
  the maturation split and `recipients` are built from the newest 20 000 rows in the
  window — when that cap bites, `meta.dailyRowCapHit` is `true` and
  `meta.dailySeriesCompleteFrom` says how far back the series is actually complete.
* `notification_history` has no index on `type` or on `createdAt` alone (only
  `(userId, createdAt)`), so this is a bounded sequential scan of the window. Review
  read, not a dashboard tile — do not poll it.
* With `?exam=`, `meta.examFilter` carries the resolved slug and the audience is matched
  through the recipient's current exam. There is **no** `examCoverage` here and none is
  needed: the source has full history. The weakness is retro-attribution, not coverage.

### ⚠️ Field-name trap: there is no `readRatePct` on `byType`

This one has already cost a reader an afternoon, so it is documented rather than left to be
rediscovered.

`readRatePct` exists **only on `byDay[]`**, where it is always a number and never null. On
`byType[]` and on `totals`, the rate field is **`maturedReadRatePct`** — because the only
honest denominator is the matured sample (see maturation, above). Read `byType[].readRatePct`
in JS or `jq` and you get `undefined` / `null` for **every** row, no matter how healthy the
data is, because the key does not exist. That looks exactly like a broken endpoint and is not
one.

`maturedReadRatePct` *is* legitimately `null` in one case and one case only:
`maturedSent === 0` — every row of that type is younger than `maturationDays` and therefore
still `pending`. Branch on `maturedSent`, not on the rate.

Verified live on 2026-07-23 with `days=60`: **every one of the four types returned a non-null
`maturedReadRatePct`.** There is no defect here.

### How to use this data

**Questions it answers.** "Which notification is worth sending?" — and much more reliably,
"which one is worth **stopping**?" It cannot answer "which one worked best", because a high
read-rate is at least as much a property of the recipient's habits as of the message (see the
`isRead` section above). Use it in one direction only.

**The decision it drives.** Turn a notification type off, re-target it, or leave it. Ranked by
**`wastedSends`**, not by rate — a 0%-read type sent 60 times is a rounding error; a
15%-read type sent 4,000 times is actively training the whole user base to ignore the bell.

**What a good visualisation is.** A kill list, and it should feel like one. `worstPerformers`
first, at the top, each as a single block: the type name, the `recommendation` sentence
rendered verbatim at readable size, and `wastedSends` as the number that carries the weight —
"1,847 of 1,869 sends went unseen" is far more persuasive than "1.2%". Below that, the full
`byType` table sorted worst-first, with `maturedSent` / `maturedRead` / `pending` visible so a
reader can see the denominator. `byDay` is a secondary sparkline at most; it is not the point.

`meta.isReadMeaning` ships inside every response precisely so the caveat cannot be separated
from the number. Put it where a reader hits it *before* quoting the figure in a deck — under
the headline, not behind an info icon three clicks away.

**The action that follows.** Stop sending the worst performer, or change its trigger. Then
re-run in 30 days.

#### What it actually returns today — measured 2026-07-23, `days=60`

| type | sent | read | maturedSent | maturedRead | **maturedReadRatePct** | pending | wastedSends | recipients |
|---|---|---|---|---|---|---|---|---|
| `trial_ending` | 1,992 | 23 | 1,869 | 22 | **1.2%** | 123 | **1,847** | 1,033 |
| `payment_success` | 13 | 10 | 13 | 10 | **76.9%** | 0 | 3 | 8 |
| `home` | 7 | 5 | 6 | 5 | **83.3%** | 1 | 1 | 6 |
| `reel` | 1 | 1 | 1 | 1 | **100%** | 0 | 0 | 1 |
| **totals** | 2,013 | 39 | 1,889 | 38 | **2.0%** | 124 | 1,851 | — |

**The highest-volume notification is, by an enormous margin, the least-read one.**
`trial_ending` is 99% of everything we send and 1.2% of it is ever seen — 1,847 wasted sends
against 1,033 distinct people. Every other type, all of them transactional and low-volume,
sits between 77% and 100%. That contrast *is* the product of this endpoint: the one message we
send at scale is the one nobody reads, and the endpoint exists to produce exactly that
kill-list line.

The live `worstPerformers` entry reads:

> **`trial_ending`** — 1,847 of 1,869 sends went unseen (1.2% read). Kill or heavily
> re-target 'trial_ending' — the volume is doing more to train people to ignore the bell than
> to bring them back.

Render that sentence, unedited. It is the deliverable.

Two honest qualifications that must travel with it, and neither of them rescues the number:

* `isRead` is a **bulk feed-open** flag, so 1.2% means "1.2% of recipients opened the in-app
  bell at any point afterwards", not "1.2% tapped this push". Push delivery is not tracked at
  all — the true tap-through could be lower, and cannot be higher in any useful sense.
* `trial_ending` goes to a **worse audience by construction** — people whose trial is expiring
  are, on average, the least engaged users in the base. Some of that 1.2% is audience, not
  copy. It does not matter for the decision: 1,847 unseen sends is 1,847 unseen sends, and the
  fix is re-targeting or a different channel, not a rewrite.

`timeToRead` came back `derivable: false` as documented — `read_at` now exists but records a
bulk feed open, and pre-migration rows are NULL. Do not synthesise it.

---

## 4. `GET /sme/analytics/onboarding-funnel`

> `auth_events` captures the **pre-auth** failures and `api_usage` covers everything
> **post-auth**, so for the first time signup → OTP → psychometric → first real
> action is traceable end to end. This endpoint is that trace.

Signup cohort in, funnel out: how many reach each stage, how many are lost between
stages, how long each stage takes where a timestamp exists — and the OTP/login
failure counts over the same window, so a spike in `otp_verify_failed` sits next to
the phone-verification drop it caused.

### ⚠️⚠️ Read this before quoting a single failure number

**`auth_events` cannot answer for time before it existed.** The table was created by
migration `20260723120000_add_auth_events` on **2026-07-23**; no row in it can
predate that. Any part of a window reaching further back is **structurally empty** —
not quiet, not healthy, *empty*.

**And it is mostly a client-ingested sink — but emitter coverage is SPLIT.**
Verified against the codebase:

| Event type | Server-side emitter | Mobile emitter | So a zero means |
| --- | --- | --- | --- |
| `otp_send_failed` | ✅ `OtpService.emitOtpSendFailed` — both `storeOtp` cache-unavailable branches **and** the Wylto delivery failure. Live from the deploy that added it. | ✅ app 2.0 | "no **backend-visible** send failure" — narrower than "no failure", but a **non-zero here is real data, do not discount it** |
| `login_attempt` | ❌ | ✅ app 2.0 | "none from a 2.0 device" — still not "no attempts" |
| `login_success` | ❌ | ✅ app 2.0 | "none from a 2.0 device" |
| `login_failed` | ❌ | ✅ app 2.0 | "none from a 2.0 device" — still not "no failures" |
| `otp_verify_failed` | ❌ | ✅ app 2.0 | "none from a 2.0 device" — still not "no failures" |

The four client-only types are **live**: app **2.0** shipped to both stores on **2026-09-17**
(Android 27, iOS 2.0 build 3) and emits all of them. They remain a **floor**, not a census,
because a device on any older build emits nothing and never will — so counts grow with 2.0
adoption, and a low number is partial coverage of the install base rather than a clean bill of
health. For `otp_send_failed` the backend also answers for itself — the exact gap that made the
Valkey outage present only as users saying "OTP expired".

(`auth_events` also carries event types this endpoint does not read — e.g. `QuotaService`
writes `chat_conversation_started`. A row in that table is not necessarily a funnel signal.)

So the response never lets a zero travel alone:

```jsonc
"authEvents": {
  "available": false,
  "captureStartedAt": "2026-07-22T18:30:00.000Z",   // IST midnight on 2026-07-23
  "windowStartsBeforeCapture": true,
  "coveredFrom": "2026-07-22T18:30:00.000Z",
  "coveredDays": 1, "uncoveredDays": 29,
  "totalEvents": 0, "totalFailures": 0,
  "zeroMeansNoDataNotNoFailures": true,             // ← branch on THIS
  "interpretation": "NO DATA YET, not \"no failures\". auth_events capture began 2026-07-23 (IST) … Emitter coverage is SPLIT … Render this panel as \"not recorded yet\"."
}
```

✅ **The `interpretation` string was corrected on 2026-09-21** (commit `4cfe6c0`). It previously
claimed the four mobile-only event types were *"pending an app release"* and would *"stay at zero
until that app release ships and users update"*. They shipped in app **2.0 on 2026-09-17**, so
that sentence was telling readers to throw away real data — the same failure mode, in the same
string, that once had people discounting genuine `otp_send_failed` counts.

The shared clause `SmeOnboardingFunnelService.interpretation()` now emits, **verbatim** (it is
also `meta.caveats[0]`, and the Swagger `@ApiOperation` twin says the same thing):

> Emitter coverage is SPLIT, and the two halves now fail in different ways. otp_send_failed has a
> SERVER-SIDE emitter (OtpService writes it on both cache-unavailable branches and on a Wylto
> delivery failure) as well as a mobile one, so its counts are backend-observed and complete: a
> non-zero is real and a zero genuinely means no backend-visible send failure.
> login_attempt/login_success/login_failed/otp_verify_failed are MOBILE-ONLY, emitted through the
> public ingest endpoint by app 2.0 (released 2026-09-17). They are therefore no longer
> structurally zero — but they are still a FLOOR, and the floor is the ADOPTED SHARE of 2.0, not
> the user base: a failure on an older build, or on any build before 2026-09-17, is not recorded
> anywhere. Weigh those four against /sme/analytics/release-health adoption before reading a
> change in them as a change in reality.

That clause is wrapped by one of three outer sentences depending on the window —
*"NO DATA — the entire requested window predates capture…"* (`coveredDays === 0`),
*"NO DATA YET, not \"no failures\"…"* (`totalEvents === 0`), or
*"N day(s) of this window are covered by auth_events capture… Counts are therefore a FLOOR on
real failures, never a ceiling."* **Render the string, do not paraphrase it** — and note the
new instruction inside it: cross-check a movement in those four counts against
`release-health` **adoption** before calling it a movement in reality.

⚠️ Whenever an emitter is added, removed, or **ships**, this string is part of the change. It has
now been stale in both directions.

* `zeroMeansNoDataNotNoFailures: true` ⇒ render **"not recorded yet"**, never
  **"no failures"**, and never a green tick.
* `byDay[].authEvents` is **`null`** — not `0` — for every IST day before capture.
  A zero there would be a lie with a number attached.
* `meta.caveats[0]` is always the `interpretation` sentence, so the warning leads
  the list rather than being buried in it.
* When the whole window predates capture the endpoint **does not even query the
  table** — it returns `coveredDays: 0` and says so.

### The five stages (each verified against `prisma/schema.prisma`)

| # | `key` | Reached when | Time-to-reach |
|---|---|---|---|
| 1 | `signup` | `user_auth."createdAt"` inside the cohort window. **This is the denominator.** | t₀ |
| 2 | `phone_verified` | `user_auth.phone_verified = true` | `phone_verified_at − createdAt`. The column is nullable, so rows verified before it existed are **counted but not timed** (`detail.verifiedWithoutTimestamp`). |
| 3 | `onboarding_completed` | `user_profiles.onboarding_completed = true` | **Not derivable** — see below |
| 4 | `psychometric_resolved` | `psychometric_test_completed = true` **OR** `psychometric_test_skipped = true` **OR** a `psychometric_test_results` row exists | `min(psychometric_test_results."createdAt") − signup`, **completions only** |
| 5 | `first_action` | The first row in the **ENGAGED union** at or after signup | `firstAction − signup` |

Stage 4 accepts three conditions on purpose: the profile flag and the results table
can disagree (a result row written before the flag existed), and the question being
answered is *"is this user past the psychometric gate?"*, not *"is this boolean
true?"*. `detail` splits `completed` / `skippedOnly` / `withResultRow`.

Stage 5 **reuses the ENGAGED union already defined in `SmeAnalyticsService`** rather
than inventing a second definition of "active" — `simulation_attempts`,
`user_content`, `user_question_attempts`, `custom_tasks` *(`source_id IS NULL` only —
rows with a source_id are auto-seeded planner content and would make everyone look
converted)*, `psychometric_test_results`, `user_document_progress`. That means
`/onboarding-funnel` and `/analytics/dau` can never disagree about what an engaged
user is.

### ⚠️ Three of the five stages have no timestamp, and the response says so

`onboarding_completed` returns:

```jsonc
"timeToReach": {
  "derivable": false,
  "reason": "NOT derivable. user_profiles has no onboarding_completed_at column; updated_at is bumped by every later profile write, so using it would be a fabrication.",
  "sample": 0, "medianSeconds": null, "p90Seconds": null
}
```

Do **not** synthesise this client-side from `updated_at`. An explicit psychometric
**skip** is likewise a bare boolean with no instant anywhere, so stage 4's median
describes completers only and `sampleCoveragePct` says what share that is.
Percentiles are **nearest-rank**, so a p90 is always a conversion that really
happened.

### ⚠️ The funnel is not strictly ordered, and that is reported, not hidden

Stages are current-state conditions, so a user can satisfy stage *n* without stage
*n−1* (onboarded without a verified phone, say). Two fields make that visible:

* `reachedWithoutPrevious` — cohort members past this stage but not the previous one;
* `droppedFromPrevious` — **raw** `previous.reached − reached`, which is therefore
  allowed to go **negative**. Clamping it to zero would hide real data drift.

### ⚠️ Conversion is measured as of NOW

The window selects **who is in the cohort by signup date**; every downstream stage is
then evaluated at request time — the same stance as `/sme/analytics/retention`. So
re-running a historical window next month can legitimately show *higher* conversion,
and signups from the last day or two drag the tail stages down because they have not
had time to convert yet. For a matured read, set an explicit `to` a few days back.

### Query

| Param | Default | Notes |
|---|---|---|
| `days` | `30` | Max **365** (`user_auth` has full history). Selects the signup cohort. |
| `from` / `to` | — | ISO datetime, or a bare `YYYY-MM-DD` read as an **IST** calendar date. The 365-day span is enforced even when both are supplied. |
| `exam` | — | Exam slug. **400** on an unknown one. Narrows the **signup cohort** by `active_exam_id` (NULL ≡ `upsc-cse`) and adds an `examSelected` block + `meta.examFilter`. ⚠️ The `authEvents` block is **not** narrowed — see below. |

### ⚠️ `?exam=` narrows the funnel but NOT the failure panel

The cohort, `stages[]`, `summary` and `byDay[]` all narrow to the exam. **`authEvents`
does not, and cannot** — a pre-auth event has no user to join an exam through, which is
the entire reason `auth_events` exists as a separate sink. So an exam-scoped funnel sits
beside an **app-wide** failure panel on the same screen. Label it explicitly, or a reader
will attribute every OTP failure in the country to one exam. The response says so in a
caveat.

Cohort attribution is also **retro-attributed**: a signup is counted under the exam the
person has picked *today*, not the one they saw on signup day. There is no switcher
history, and the caveat says that too.

### `examSelected` — present only with `?exam=`, and deliberately NOT a sixth stage

`stages[]` is a published array that the portal indexes **positionally** and this doc
enumerates by order. Slipping an exam-selection stage into the middle of it would
silently renumber every existing stage — for a caller who never asked about exams. So it
sits **beside** the funnel as its own key:

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/analytics/onboarding-funnel?days=14&exam=appsc-group-1` — the `examSelected`
key of that response, verbatim (staging's signup volume is tiny, which is why the numbers
are small; the keys are the point):

```jsonc
"examSelected": {
  "exam": "appsc-group-1",
  "cohortSignups": 4,
  "windowSignups": 5,
  "explicitlySelected": 5,
  "explicitlySelectedPct": 100,
  "nullCount": 0,
  "nullBucketedAs": "upsc-cse",
  "byExam": [
    { "examId": "appsc-group-1", "users": 4 },
    { "examId": "upsc-cse", "users": 1 }
  ],
  "note": "4 of 5 signups in this window belong to \"appsc-group-1\"; the rest of this response describes only those. 5 signup(s) carry an explicit active_exam_id — app 2.0 (released 2026-09-17) made the exam picker onboarding step 1, so this share rises with 2.0 adoption. The remaining 0 never picked (pre-2.0 onboarding, switcher never used) and are counted under upsc-cse, which is the exam they were actually served."
}
```

The same response's `meta.examFilter` is `"appsc-group-1"`, and its last caveat — also
verbatim — is the retro-attribution warning in words:

> *"Exam filter \"appsc-group-1\": the COHORT is narrowed by `user_profiles.active_exam_id`
> read AS OF NOW (NULL ≡ upsc-cse — the pre-2.0 remainder, who were served the default). A
> signup is therefore attributed to the exam the person has picked today, not the one they
> saw on signup day; there is no history of the switcher. The `auth_events` block below is
> NOT narrowed at all — a pre-auth event has no user to join an exam through — so it stays
> APP-WIDE beside an exam-scoped funnel."*

Three things about it that will otherwise be read as bugs:

* **`byExam` describes the WHOLE window, not the filtered cohort.** A breakdown of a set
  already narrowed to one exam is a single bar, which answers nothing. `cohortSignups`
  and `windowSignups` sit side by side precisely so the panel can say *"96 of 412 signups
  in this window are yours"*.
* **`nullCount` is folded into `upsc-cse` inside `byExam`** — exactly as every other
  per-exam number in this API does it — and reported separately beside it. When it is
  large, "UPSC" partly means "unclassified". `nullBucketedAs` states the rule rather than
  implying it.
* **`explicitlySelectedPct` is an adoption metric, not a funnel metric.** It measures the
  share of signups that came through 2.0's picker (or have used the switcher); it rises
  with 2.0 uptake, not with product quality. `cohort.signups` still equals
  `examSelected.cohortSignups` — the funnel's own denominator is unchanged.

### Response (abridged)

```jsonc
{
  "cohort": { "from": "…", "to": "…", "windowDays": 30, "signups": 412, "capped": false, "cap": 20000 },
  "summary": {
    "cohortSignups": 412,
    "biggestDropStage": "onboarding_completed",
    "biggestDropLabel": "Onboarding completed",
    "biggestDropCount": 137, "biggestDropPct": 41.5,
    "endToEndConversionPct": 38.1,
    "note": "137 of 412 signups (41.5% of the previous stage) are lost at \"Onboarding completed\" — the largest single drop. End to end, 38.1% of signups reach a genuine product action."
  },
  "stages": [{
    "key": "phone_verified", "label": "Phone verified", "order": 2,
    "definition": "user_auth.phone_verified = true (current state).",
    "reached": 330, "reachedPctOfSignupsPct": 80.1,
    "conversionFromPreviousPct": 80.1, "droppedFromPrevious": 82, "dropOffPct": 19.9,
    "reachedWithoutPrevious": 0,
    "timeToReach": {
      "derivable": true, "reason": null, "sample": 328, "sampleCoveragePct": 99.4,
      "medianSeconds": 41.5, "p90Seconds": 512, "medianLabel": "42s", "p90Label": "8.5m"
    },
    "detail": { "verifiedWithoutTimestamp": 2 }
  }],
  // "examSelected": { … }   ← only with ?exam=, a SIBLING of stages[], never inside it
  "byDay": [{
    "day": "2026-07-24", "signups": 18, "phoneVerified": 4,
    "onboardingCompleted": 3, "psychometricResolved": 3, "firstAction": 2,
    "phoneVerifiedPct": 22.2,
    "authEvents": { "loginAttempt": 210, "loginSuccess": 150, "loginFailed": 12, "otpSendFailed": 3, "otpVerifyFailed": 47 }
  }],
  "authEvents": {
    "available": true, "totalEvents": 430, "totalFailures": 80,
    "byType": [{ "eventType": "otp_verify_failed", "count": 30, "distinctIdentifiers": 22, "isFailure": true }],
    "zeroMeansNoDataNotNoFailures": false,
    "interpretation": "…",
    "otpFailureRate": { "derivable": false, "reason": "…no otp_sent / otp_verify_success counterpart…" },
    "loginSuccessRatePct": 75,
    "correlation": {
      "derivable": true, "minSignupsPerDay": 5,
      "worstDay": { "day": "2026-07-24", "signups": 18, "phoneVerified": 4, "phoneVerifiedPct": 22.2, "loginFailed": 12, "otpSendFailed": 3, "otpVerifyFailed": 47 },
      "medianOtpVerifyFailedPerDay": 2,
      "note": "2026-07-24 (IST) had the worst phone-verification conversion: 4/18 (22.2%). That day saw 47 otp_verify_failed and 3 otp_send_failed events, against a per-day median of 2 otp_verify_failed — a spike worth chasing."
    },
    "dailyRowCapHit": false, "identifierRowCapHit": false
  },
  "meta": { "conversionMeasuredAsOf": "…", "stageDefinitions": { … }, "cohortRowCapHit": false,
            "examFilter": "appsc-group-1",   // ← only with ?exam=
            "caveats": [ … ] }
}
```

`authEvents.correlation` is the whole reason this endpoint is worth building: it
names the covered IST day with the **worst phone-verification conversion**, puts that
day's OTP/login failure counts beside it, and compares them to the per-day median so
you can tell a spike from background noise. It needs ≥5 signups on a day before it
will name one — below that it says `derivable: false` rather than pointing at a day
where two people happened not to verify.

### ⚠️ There is no OTP failure *rate*

`AUTH_EVENT_TYPES` has **no `otp_sent` and no `otp_verify_success`**, so an OTP
failure count has no denominator — nothing records how many OTPs were sent or
verified successfully. The response returns
`otpFailureRate: { derivable: false, reason: … }`. **Do not synthesise one
client-side.** `login_attempt` → `login_success` *is* a real pair, which is the only
reason `loginSuccessRatePct` exists.

### Caveats

* `byDay[].authEvents` is **app-wide** for that IST day, not restricted to that day's
  signup cohort — a pre-auth failure has no user id to restrict by. Read it as a
  same-day environmental signal, not as "these signups failed".
* `psychometric_test_results` is a **member** of the ENGAGED union, so a user whose
  only action was the psychometric test satisfies **stage 5 with the very row that
  satisfied stage 4**. `stages[4].detail.bySource` shows how many — if that source
  dominates, the "first real action" number is thinner than it looks.
* An action timestamped **before** its account existed is dropped rather than counted
  as an instant conversion; same for a `phone_verified_at` that precedes signup.
* `auth_events` rows are purged after **90 days**, so a window longer than that loses
  its own tail on top of the capture-start floor.
* Caps: 20 000 cohort rows (newest signups first), 25 000 first-action rows per
  ENGAGED source, 20 000 `auth_events` rows for the day series, 20 000 distinct
  identifier rows. Each one reports itself in `meta` when it bites; `byType` totals
  are exact (a DB-side aggregate) even when the day series is capped.
* With `?exam=`: the cohort is narrowed by `active_exam_id` **read as of now** (NULL ≡
  `upsc-cse`), so a signup is attributed to the exam the person has picked today, not
  the one they saw on signup day. The **`authEvents` block is not narrowed at all** and
  stays app-wide beside the exam-scoped funnel. Both sentences ship as caveats.

### How to use this data

**Questions it answers.** "Of everyone who signed up this month, how many ever used the
product?" · "Which single step loses the most people?" · "How long does each step take?" ·
"Was a bad day caused by OTP failures, or by something else?"

**The decision it drives.** What to fix next in the top of the funnel — and, just as often,
what *not* to fix. `summary.biggestDropStage` names the stage; `summary.note` is a
ready-to-paste sentence. This is the one endpoint of the four whose output should change a
roadmap.

**What a good visualisation is.** A vertical funnel where the **losses** are the visual
subject, not the survivors. Five stage rows, each showing `reached` and
`reachedPctOfSignupsPct`, with the drop between rows rendered as an explicit, labelled gap —
"−389 (21%)" — sized to the loss. The biggest drop gets emphasis automatically because it is
the biggest gap. Beside each stage, `timeToReach.medianLabel` / `p90Label` where derivable,
and the literal words "not derivable" where not — `onboarding_completed` has no timestamp
column and never will until the schema changes, and inventing one from `updated_at` would be
a fabrication.

Do **not** render this as five equal-size stat cards in a row. A funnel whose stages are all
the same size communicates nothing; the entire information content is in the differences.

**The action that follows.** Instrument or fix the biggest drop, then re-run with an explicit
`to` a few days back so the tail stages are matured.

#### What it actually returns today — measured 2026-07-23 ~14:44 UTC, `days=30`

| # | stage | reached | % of signups | conv. from prev | dropped | median time |
|---|---|---|---|---|---|---|
| 1 | Signed up | **2,057** | 100% | — | — | t₀ |
| 2 | Phone verified | **1,853** | 90.1% | 90.1% | −204 | 29s (p90 82s) |
| 3 | Onboarding completed | **1,464** | 71.2% | 79.0% | **−389** | *not derivable* |
| 4 | Psychometric resolved | **1,478** | 71.9% | 101% | −14 | 6.9m (p90 13.8m) |
| 5 | First genuine product action | **1,100** | 53.5% | 74.4% | **−378** | 6.9m (p90 27m) |

**About half of everyone who signs up never takes a single product action.** End-to-end
conversion is 53.5%. The two biggest losses are **onboarding completion (−389)** and
**psychometric-resolved → first action (−378)**, and they are roughly equal — which matters,
because it means there is no single villain: fixing onboarding alone recovers at most half the
gap.

Phone verification, by contrast, is not the problem and should not be optimised: 90.1% pass it
with a **median of 29 seconds** and a p90 of 82 seconds. OTP delivery is working.

Four things in that payload that a naive UI will get wrong:

* **Stage 4 has a *negative* drop (−14) and 101% conversion.** That is not a bug and must not
  be clamped. Stages are current-state booleans, not an ordered journey — 14 users resolved
  the psychometric gate without ever flipping `onboarding_completed`. The response reports
  exactly that as `reachedWithoutPrevious: 14`. Render the negative honestly and put
  `reachedWithoutPrevious` next to it as the explanation; a funnel that silently clamps to
  zero is hiding real data drift.
* **Stage 5 is thinner than 1,100 looks.** `detail.bySource` was
  `psychometric_test_results: 991`, `user_content: 48`, `user_question_attempts: 33`,
  `simulation_attempts: 17`, `custom_tasks: 11` — so **991 of the 1,100 "first real actions"
  are the psychometric test itself**, the same row that satisfied stage 4. Only ~109 users did
  anything else. Show `bySource` on the stage-5 row; without it the headline is misleading, and
  the doc's own caveat says so.
* **Stage 4's median describes completers only.** `sampleCoveragePct: 67.1` — 991 completed
  (timed) versus 487 who explicitly skipped (countable, untimeable, because a skip is a bare
  boolean with no instant stored anywhere). Print the coverage percentage next to the median.
* **`timeToReach.derivable: false` on stage 3 is permanent** until `user_profiles` gets an
  `onboarding_completed_at` column (§7). Show the `reason` string; do not leave a blank cell,
  and never fall back to `updated_at`.

**The `authEvents` block, honestly.** On that run: `available: true`, `coveredDays: 1`,
`uncoveredDays: 29`, `totalEvents: 4` — two `login_attempt`, one `login_success`, one
`login_failed`, and **zero** `otp_send_failed` / `otp_verify_failed`. Four events is not a
signal; capture began on 2026-07-23 and 29 of the 30 days in the window structurally cannot
contain a row. `byDay[].authEvents` is `null` for all 29 of them, not `0`. The correlation
block *did* fire (`derivable: true`) and named 2026-07-23 as the worst verification day
(36/45, 80%) while explicitly concluding *"no spike, so the drop is probably not an OTP
delivery problem"* — which is the block working correctly: it is designed to rule causes
**out** as readily as in.

Render the whole panel as **"capture started 2026-07-23 — 1 of 30 days covered"**, with the
counts subordinate to that sentence. It stops being an empty panel and starts being real data
in late August; until then a green tick next to "0 OTP failures" would be a lie with a number
attached.

⚠️ **These integers are a snapshot of a rolling window, not constants.** A run twenty minutes
earlier on the same day returned 2,059 → 1,855 → 1,466 → 1,480 → 1,101. The cohort is "signups
in the last 30 days" and conversion is evaluated **as of now**, so every re-run moves. Never
cache a funnel number as a fact; always show `meta.conversionMeasuredAsOf`.

---

## 5. `GET /sme/analytics/purchase-funnel` — new 2026-09-21

> Every other analytics read answers *"what is happening"*. This one answers **"where
> does the money stop"** — and it is the only read that joins the **client's** view of a
> purchase (`auth_events`) to the **server's** (`orders`).

**Field-level contract:** [SME_USAGE_ANALYTICS.md §4c](./SME_USAGE_ANALYTICS.md) — that
doc is the reference and carries the full example body. This section is the narrative:
why it exists, and the three ways it will be misread.

`GET /sme/analytics/purchase-funnel?days=30&exam=` returns, **per IST day**:
`paywallViewed → upgradeTapped → checkoutOpened → checkoutAbandoned / purchaseFailed →
ordersPaid`, plus `conversionPct`, a `totals` block and a top-level `caveats: string[]`
(this is the one endpoint here with no `meta` wrapper). `days` is **1..90**, default 30 —
`auth_events` is purged at 90, so a longer window would lose its own tail.

### Why it is its own endpoint, and not another metric on `/summary`

The two halves of a purchase have **genuinely different trust** and **genuinely
different exam attribution**, and keeping them in one place is what stops the difference
being papered over:

| | The five funnel steps | `ordersPaid` |
|---|---|---|
| Written by | the **app**, through the *public* diagnostics ingest endpoint | the **server**, on a verified payment |
| Trust | a floor; nothing here is forgery-proof, and a build that does not emit is invisible | server truth |
| Exists since | app **2.0**, 2026-09-17 — and only on adopted devices | always |
| Exam | the user's **current** `active_exam_id` (retro-attributed) | `orders.exam_id` — the exam the money actually bought, exact |

### Two choices that matter more than the query

**1. DISTINCT USERS, not event rows.** A paywall re-renders; a checkout sheet gets
reopened. Counting rows would make `paywallViewed` a function of how chatty the build is,
and `conversionPct` would *fall* every time the UI got more responsive. Each step is
`count(DISTINCT user_id)` inside its IST day. The corollary is that window `totals` are
those per-day distinct counts **summed** — a user active on three days counts three
times. That is the only total consistent with the chart above it.

**2. `conversionPct` is `null`, never `0`, on a zero denominator.** A `0%` there reads as
*"we showed the paywall and converted nobody"*. The truth is that **there is no
denominator**, which is a different fact and usually means the day predates the emitting
build rather than that the paywall failed. Branch on `paywallViewed`, not on the rate.

### The three misreadings, in advance

* **`conversionPct > 100` is not a bug.** `ordersPaid` is not joined to the events, so
  someone who bought without a recorded paywall view lands in the numerator only. A
  conversion above 100% is a direct **measure of emitter coverage** — the gap between how
  many people bought and how many the client told us saw the paywall. Render it, do not
  clamp it. As 2.0 adoption rises it will fall toward a real rate, and *that fall is not
  a regression*.
* **A rising conversion can be adoption, not improvement.** Same cause, opposite
  direction: every step is a floor whose height is 2.0's adopted share. Cross-check
  against `release-health` adoption before crediting a paywall change.
* **A per-exam `conversionPct` mixes two attribution rules.** `auth_events` rows carry no
  exam, so with `?exam=` a step is attributed to the exam the user has picked **at query
  time** — a user who viewed the APPSC paywall in June and has since switched to UPSC
  counts today as a UPSC paywall view. `ordersPaid` does not share this weakness. So the
  per-exam rate has a **retro-attributed denominator** and an **exactly-attributed
  numerator**: read it as directional, and read the all-exams number (omit `exam`) as the
  exact one. The service ships exactly that sentence as the last `caveats` entry whenever
  `exam` is supplied.

### What a good visualisation is

A **per-day stacked/step column** for the five client steps with `ordersPaid` as a
visually distinct mark — it is a different *kind* of fact, not a sixth step — and
`conversionPct` as a **line on a secondary axis**. The line must show a **gap** where the
value is `null` and must be allowed past **100**. The step-to-step loss is the subject:
`paywallViewed → upgradeTapped` is the interest drop, `checkoutOpened →
checkoutAbandoned` is the pricing/friction drop, and `purchaseFailed` beside them is the
one that is *ours to fix*.

Render `caveats[]` on the panel. Six strings always, seven with `?exam=`.

**The action that follows.** If the loss is at `upgradeTapped`, the paywall copy or price
is the problem. If it is at `checkoutAbandoned`, the checkout is. If `purchaseFailed` is
non-trivial, that is an engineering bug and it belongs in the tracker, not in a growth
review.

---

## 6. Performance stance (shared by all five)

These reads share the **production database the mobile app runs on**. So:

* every query is bounded by a **date window** with a hard max, enforced even when the
  caller supplies an explicit `from`/`to`;
* every aggregation carries a **row cap**, and the response reports when the cap bit
  rather than silently truncating;
* thresholds are pushed into SQL (`HAVING`) wherever possible so the DB, not Node,
  discards the tail;
* §1–§4 are written through the **Prisma query builder**, not `$queryRaw` —
  column casing in this schema is mixed *per table*
  (`simulation_attempts."userId"/"startedAt"` quoted camelCase vs
  `user_question_attempts.user_id/created_at` snake_case) and a raw-SQL typo is a
  runtime error, not a compile error. The one thing the builder cannot express is
  IST day-bucketing (`date_trunc`), which is done in TS with the shared
  `common/utils/ist-date.util` helper under a row cap.
  **§5 is the exception**: `purchase-funnel` needs `count(DISTINCT user_id)` grouped by
  an IST-shifted date, which the builder cannot express at all, so it uses two
  `$queryRaw` aggregates against `auth_events` and `orders` — both single-table, both
  snake_case, and the exam slug is **always a bind parameter**, never interpolated. It
  has no row cap because the DB returns at most `days × 5` rows.

**None of these are dashboard tiles.** They are review reads. Do not poll them.

---

## 7. Known gaps worth fixing (not fixed here — schema changes)

| Gap | Impact | Fix |
|---|---|---|
| `notification_history.read_at` exists *(done, `c146d94`)* but records a **bulk feed open**, and pre-migration rows are NULL | time-to-read is still not quotable per notification | let post-migration history accumulate, then surface `read_at − sent_at` as "time until the recipient came back", never as "time to open this push"; a real per-notification metric needs an open event |
| `notification_history` has no index on `(type, createdAt)` | §3 is a sequential scan | add the index |
| `user_question_attempts` has no index on `updated_at`/`created_at` | §1 content pool is a sequential scan | add `@@index([updatedAt])` |
| `simulation_answers` has no timestamp | §1 must fan out through `simulation_attempts` | add `createdAt` |
| `simulation_attempts` has no index on `startedAt` | that fan-out is a sequential scan | add `@@index([startedAt])` |
| Question quality does not read the exam dimension | the columns exist (`content_documents.examIds`; simulation questions via `simulations.examIds`) but §1 ignores them, so a mixed-exam pool is reported as one bank | **still open after `4cfe6c0`** — that commit added `?exam=` to all 17 pre-existing `/sme/analytics/*` routes (18 with §5), but `/sme/content/question-quality` is on a different controller and was untouched. Add the same filter through `examArraySql`. |
| **`auth_events` has no exam column and no relation to join one through** | three per-exam numbers cannot be honest: §2's `clientErrors` stays app-wide (`meta.clientErrorsExamScoped: false`), §4's whole `authEvents` block stays app-wide, and §5's five funnel steps have to borrow the user's *current* exam | a nullable `exam_id` on `auth_events`, stamped by the ingest endpoint from the `X-Exam` header. Only fixes it going forward — the same deploy-day floor as `api_usage.exam_id`. |
| **No history of `user_profiles.active_exam_id`** | every person-keyed per-exam number is **retro-attributed**: a user, and everything they ever did, is counted under the exam they picked *today*. Worst on a newly-launched exam, where switchers are a large share of the population | an append-only `user_exam_switch` log, and read the exam *as of* the row's own timestamp. Until then the caveat is the fix. |
| **`api_usage.exam_id` starts 2026-09-21** | there is **no per-exam usage history**, only a floor that begins on deploy day. `examCoverage.since` reports it rather than hiding it | nothing to fix — it ages out on its own. Do not backfill it by guessing `upsc-cse`; a NULL there genuinely means "not recorded". |
| **`user_profiles` has no `onboarding_completed_at`** | §4 stage 3 time-to-reach is permanently underivable — the single biggest hole in the funnel | nullable `onboarding_completed_at`, stamped where `onboarding_completed` is first set to true |
| **`user_profiles` has no `psychometric_skipped_at`** | §4 can count skippers but never time them | nullable `psychometric_skipped_at` |
| **No `otp_sent` / `otp_verify_success` event type** | §4 can report OTP failure *counts* but never a *rate* | add both to `AUTH_EVENT_TYPES` and emit them from the client alongside the failures |
| **No server-side emitter for `login_*` / `otp_verify_failed`** *(`otp_send_failed` is DONE — `OtpService.emitOtpSendFailed`)* | app 2.0 now emits all four from the client, but the signal is still capped at the 2.0-adopted share of the install base and can never cover older builds | emit `otp_verify_failed` and the `login_*` pair from the backend auth path too, so no part of the signal depends on an app version |
| **`user_auth` has no index on `createdAt`** | §4's cohort read is a sequential scan of `user_auth` | add `@@index([createdAt])` |
| `user_content` / `user_document_progress` have no `created_at`-leading index | §4's first-action group-bys are bounded sequential scans | add `@@index([createdAt])` to each |

---

Related: [SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) ·
[SME_ANALYTICS_FRONTEND_GUIDE.md](./SME_ANALYTICS_FRONTEND_GUIDE.md) ·
[SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) (the per-user view of the same
`api_usage` / `auth_events` tables) ·
[SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) (how the sends in §3 are made) ·
[archive/WHAT_CHANGED_2026-07-23.md](./archive/WHAT_CHANGED_2026-07-23.md) (archived)
