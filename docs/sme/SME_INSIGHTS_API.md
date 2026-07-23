# SME — Insight Endpoints (question quality · release health · notification effectiveness · onboarding funnel)

Four read-only decision endpoints for the SME portal. Each one exists because a
decision is currently being made by guessing:

| Endpoint | The guess it replaces |
|---|---|
| `GET /sme/content/question-quality` | "which questions are bad?" — today the content team eyeballs it |
| `GET /sme/analytics/release-health` | "should we force-update?" — today it's a hunch |
| `GET /sme/analytics/notification-effectiveness` | "is this push working?" — today the only lever is *send more* |
| `GET /sme/analytics/onboarding-funnel` | "where do signups die?" — today nobody can answer at all |

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
behind an "i" on the panel rather than reprinting it in a deck.

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
  "meta": { "pools": ["content","simulation"], "minAttempts": 20, "candidateCapHit": false, "caveats": [ … ] }
}
```

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
* **There is no "exam" dimension** on either bank (`content_documents` and
  `simulation_questions` both carry subject/topic only), so filtering by exam is not
  possible today.
* Neither attempts table has a **timestamp index**, so this is a bounded sequential
  scan of the window. It is a review read, not a dashboard tile — do not poll it.

---

## 2. `GET /sme/analytics/release-health`

> `AppConfig.minAppVersion` already provides the force-update **mechanism**. This
> supplies the **number**.

Per **app version × platform**: active users, requests, 4xx/5xx error rate,
`client_error` count per active user, and adoption share — plus an explicit
comparison against the next-older build on the same platform, so *"is 1.7 worse
than 1.6?"* is one glance.

### ⚠️ `app_version IS NULL` is the pre-1.7 cohort, not missing data

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
    "platform": "ios", "activeUsers": 577, "requests": 141002, "versions": 3,
    "worstVersion": null, "worstVersionLabel": null, "worstVersionErrorRatePct": null,
    "preHeaderSharePct": 8.3
  }],
  "meta": { "windowDays": 14, "apiUsageRawRetentionDays": 30, "windowExceedsApiUsageRetention": false, "caveats": [ … ] }
}
```

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

**And it is mostly a client-ingested sink — but emitter coverage is now SPLIT.**
Verified against the codebase:

| Event type | Server-side emitter | Mobile emitter | So a zero means |
| --- | --- | --- | --- |
| `otp_send_failed` | ✅ `OtpService.emitOtpSendFailed` — both `storeOtp` cache-unavailable branches **and** the Wylto delivery failure. Live from the deploy that added it. | pending a release | "no **backend-visible** send failure" — narrower than "no failure", but a **non-zero here is real data, do not discount it** |
| `login_attempt` | ❌ | pending a release | "not recorded yet" |
| `login_success` | ❌ | pending a release | "not recorded yet" |
| `login_failed` | ❌ | pending a release | "not recorded yet" |
| `otp_verify_failed` | ❌ | pending a release | "not recorded yet" |

So for the four mobile-only types: until that app release ships and users update,
**these counts stay at zero however many logins are actually failing in
production.** For `otp_send_failed` the backend now answers for itself — the exact
gap that made the Valkey outage present only as users saying "OTP expired".

(Other server-side writers exist for event types this endpoint does not read:
`QuotaService` → `chat_conversation_started`, `DifyMetricsService` → `dify_call`.)

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
  "interpretation": "NO DATA YET, not \"no failures\". auth_events capture began 2026-07-23 (IST) … Emitter coverage is SPLIT. otp_send_failed now has a SERVER-SIDE emitter … login_attempt/login_success/login_failed/otp_verify_failed remain MOBILE-ONLY … Render this panel as \"not recorded yet\"."
}
```

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
  "meta": { "conversionMeasuredAsOf": "…", "stageDefinitions": { … }, "cohortRowCapHit": false, "caveats": [ … ] }
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

---

## 5. Performance stance (shared by all four)

These reads share the **production database the mobile app runs on**. So:

* every query is bounded by a **date window** with a hard max, enforced even when the
  caller supplies an explicit `from`/`to`;
* every aggregation carries a **row cap**, and the response reports when the cap bit
  rather than silently truncating;
* thresholds are pushed into SQL (`HAVING`) wherever possible so the DB, not Node,
  discards the tail;
* everything is written through the **Prisma query builder**, not `$queryRaw` —
  column casing in this schema is mixed *per table*
  (`simulation_attempts."userId"/"startedAt"` quoted camelCase vs
  `user_question_attempts.user_id/created_at` snake_case) and a raw-SQL typo is a
  runtime error, not a compile error. The one thing the builder cannot express is
  IST day-bucketing (`date_trunc`), which is done in TS with the shared
  `common/utils/ist-date.util` helper under a row cap.

**None of these are dashboard tiles.** They are review reads. Do not poll them.

---

## 6. Known gaps worth fixing (not fixed here — schema changes)

| Gap | Impact | Fix |
|---|---|---|
| `notification_history.read_at` exists *(done, `c146d94`)* but records a **bulk feed open**, and pre-migration rows are NULL | time-to-read is still not quotable per notification | let post-migration history accumulate, then surface `read_at − sent_at` as "time until the recipient came back", never as "time to open this push"; a real per-notification metric needs an open event |
| `notification_history` has no index on `(type, createdAt)` | §3 is a sequential scan | add the index |
| `user_question_attempts` has no index on `updated_at`/`created_at` | §1 content pool is a sequential scan | add `@@index([updatedAt])` |
| `simulation_answers` has no timestamp | §1 must fan out through `simulation_attempts` | add `createdAt` |
| `simulation_attempts` has no index on `startedAt` | that fan-out is a sequential scan | add `@@index([startedAt])` |
| No "exam" dimension on either question bank | cannot filter question-quality by exam | out of scope; product decision |
| **`user_profiles` has no `onboarding_completed_at`** | §4 stage 3 time-to-reach is permanently underivable — the single biggest hole in the funnel | nullable `onboarding_completed_at`, stamped where `onboarding_completed` is first set to true |
| **`user_profiles` has no `psychometric_skipped_at`** | §4 can count skippers but never time them | nullable `psychometric_skipped_at` |
| **No `otp_sent` / `otp_verify_success` event type** | §4 can report OTP failure *counts* but never a *rate* | add both to `AUTH_EVENT_TYPES` and emit them from the client alongside the failures |
| **No server-side emitter for `login_*` / `otp_verify_failed`** *(`otp_send_failed` is DONE — `OtpService.emitOtpSendFailed`)* | §4's login signal is blind until a mobile release ships | emit `otp_verify_failed` and the `login_*` pair from the backend auth path too, so no part of the signal depends on an app release |
| **`user_auth` has no index on `createdAt`** | §4's cohort read is a sequential scan of `user_auth` | add `@@index([createdAt])` |
| `user_content` / `user_document_progress` have no `created_at`-leading index | §4's first-action group-bys are bounded sequential scans | add `@@index([createdAt])` to each |

---

Related: [SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) ·
[SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) ·
[SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md)
