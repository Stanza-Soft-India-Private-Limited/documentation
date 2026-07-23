# SME — Insight Endpoints (question quality · release health · notification effectiveness)

Three read-only decision endpoints for the SME portal. Each one exists because a
decision is currently being made by guessing:

| Endpoint | The guess it replaces |
|---|---|
| `GET /sme/content/question-quality` | "which questions are bad?" — today the content team eyeballs it |
| `GET /sme/analytics/release-health` | "should we force-update?" — today it's a hunch |
| `GET /sme/analytics/notification-effectiveness` | "is this push working?" — today the only lever is *send more* |

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

`notification_history` has **no `read_at` column** and no per-row mark-read path.
The instant of reading is never stored. So the response returns:

```jsonc
"timeToRead": {
  "derivable": false,
  "reason": "Not derivable. notification_history has no read_at column and no per-row mark-read path — isRead is flipped in bulk by markFeedSeen, so the instant of reading is never stored. Any \"time to read\" number would be fabricated. …"
}
```

Do **not** synthesise this client-side. Getting it would need a nullable `read_at`
(stamped inside the same `updateMany`) or a per-notification open event — both are
schema changes, and neither is part of this endpoint.

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

## 4. Performance stance (shared by all three)

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

## 5. Known gaps worth fixing (not fixed here — schema changes)

| Gap | Impact | Fix |
|---|---|---|
| `notification_history` has no `read_at` | time-to-read is permanently underivable | nullable `read_at`, stamped in the same `markFeedSeen` `updateMany` |
| `notification_history` has no index on `(type, createdAt)` | §3 is a sequential scan | add the index |
| `user_question_attempts` has no index on `updated_at`/`created_at` | §1 content pool is a sequential scan | add `@@index([updatedAt])` |
| `simulation_answers` has no timestamp | §1 must fan out through `simulation_attempts` | add `createdAt` |
| `simulation_attempts` has no index on `startedAt` | that fan-out is a sequential scan | add `@@index([startedAt])` |
| No "exam" dimension on either question bank | cannot filter question-quality by exam | out of scope; product decision |

---

Related: [SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) ·
[SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) ·
[SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md)
