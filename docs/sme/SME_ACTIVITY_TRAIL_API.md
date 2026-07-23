# SME — User Activity Trail API

The support/debug surface that answers **"what happened to this user, end to end"**.
Five endpoints under `/sme/users/*`: a snapshot header, a merged timeline, auto-grouped
incidents, staff notes, and support-code lookup.

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Auth:** `x-api-key: <API_KEY_SECRET>` header on **every** request (`@SmeApiKey`).
**Swagger:** `/api/docs`, group **SME** — live field list, source of truth if this doc drifts.
**Timezone:** the server runs UTC; every IST-labelled field (`istDate`, `startedAtIST`) and
every `YYYY-MM-DD` date input is resolved at **UTC+05:30**. `Date` fields are ISO-8601 UTC.

---

## ⛔ Read this before you write a single line of client code

### Success responses are RAW. There is no `{ success, data }` envelope.

`ResponseInterceptor` **exists** in `src/common/interceptors/response.interceptor.ts` and is
**never registered**. The only global `APP_INTERCEPTOR` in `app.module.ts` is
`ApiUsageInterceptor`, which writes usage rows and does not touch the body. So a 200 from
`GET /sme/users/:id/snapshot` is the snapshot object *itself* — `response.user.email`, not
`response.data.user.email`.

> This exact wrong assumption — reading `.data` off a raw body — crashed a mobile release
> last week. Do not port an envelope-unwrapping helper from another integration. If some
> other doc in this folder (e.g. `SME_FEEDBACK_API.md`) says "all responses ride the global
> envelope", that statement is wrong; the code is the authority.

### Error responses DO have a different shape.

Errors go through `AllExceptionsFilter` (`src/common/filters/exception.filter.ts`, registered
via `app.useGlobalFilters` in `main.ts`), which emits:

```json
{
  "success": false,
  "message": "User not found",
  "error": "Not Found",
  "statusCode": 404,
  "timestamp": "2026-07-23T09:14:22.108Z",
  "path": "/api/v1/sme/users/abc/snapshot",
  "method": "GET"
}
```

So: **branch on HTTP status, not on the presence of `success`.** A success body has no
`success` key at all; an error body always does. Client rule that works for every endpoint
here — `if (res.ok) use body; else read body.message`.

| Status | When | `message` |
|---|---|---|
| 200 | snapshot / timeline / incidents / support-code lookup | — |
| 201 | note created | — |
| 400 | bad `sources` value, malformed cursor, `from` after `to`, unusable support code | explains exactly what was rejected |
| 401 | missing/invalid `x-api-key` | `Invalid or missing API key` |
| 404 | unknown user id, or no user matches the support code | `User not found` / `No user matches support code …` |
| 409 | support code matched more than one user | `Support code … is ambiguous` |

⚠️ **Known filter limitation on 409:** the service throws
`ConflictException({ message, candidates: [...] })`, but `AllExceptionsFilter` only copies
`message` and `error` out of the exception body (the only structured pass-through it has is
for 402 quota fields). **`candidates` does not reach the client.** Treat a 409 as "ambiguous,
ask the user for their email/phone instead" — do not build a candidate-picker UI, there is
nothing to populate it with. (2^40 code space over a few thousand users makes this
effectively unreachable in practice.)

---

## What this data can and cannot tell you

Read this section before designing anything on top of the endpoints. Every limitation below
is structural, not a bug to be fixed later.

**1. `api_usage` history starts when the capture interceptor shipped — and is purged at 30 days.**
There is no request trail before the interceptor's deploy date, and
`ApiUsageRollupService` purges raw `api_usage` rows older than **30 days** nightly (01:30 IST).
So even though the timeline window maxes out at 90 days, the `api_usage` source is
effectively a **rolling 30-day** view. Do not promise users a window. Check what actually
exists before you claim one:

```sql
SELECT min(created_at), max(created_at), count(*) FROM api_usage;
```

Companion retention, same job: `auth_events` **90 days**, `api_usage_daily` rollup **120 days**.
Every other source (orders, attempts, notifications, notes…) is untouched by the purge and
goes back to account creation — the 90-day *query window* is the only limit there.

**You do not have to hardcode any of this.** Every windowed response carries a `retention`
object — on `meta` for the timeline and incidents, on `activity` for the snapshot — so the UI
can say it out loud instead of the agent inferring that a quiet stretch means an idle user:

```json
"retention": {
  "apiUsageRawRetentionDays": 30,
  "apiUsageCompleteFrom": "2026-06-23T09:14:22.000Z",
  "windowExceedsApiUsageRetention": true,
  "note": "Raw api_usage rows are purged after 30 days, so request activity before … is gone — absence there is retention, not inactivity. Other sources cover the full window."
}
```

`note` is `null` (and the boolean `false`) whenever the requested window fits inside retention.
When it is set, render it — a `days=90` request is silently empty for `api_usage` past day 30.

**2. Chat content lives only in Dify. We store a pointer, not a transcript.**
On a **successful** `POST /quota/finalize` for a **chat** feature (`chat_mentor`, `chat_sme` or
the legacy `chat`) carrying a `conversationId`, the server writes one `auth_events` row
`eventType = 'chat_conversation_started'` with `metadata = { feature, conversationId }`. Both
gates matter: the one-shot workflow features (`mains_eval`, `flashcard`, `mnemonic`,
`pyq_variation`) and the failure path (`success: false`, which refunds the token) write
**nothing**, so a row here always means a chat conversation really started. That
row proves a conversation happened, when, and on which feature — and gives you the Dify id to
look it up. **It does not contain a single message.** Never label this panel "chat history".

**3. Quota is Redis with a midnight-IST TTL. The snapshot is TODAY, and only today.**
There is no quota history table anywhere. Yesterday's counters were deleted by the TTL, not
hidden from you. The response says `scope: "today_only"` and `resetsAtISTMidnight: true`
precisely so the portal never renders a quota series. When the read throws, `available` goes
`false` with an `unavailableReason` — render that, not zeroes.

⚠️ **`available: true` is not a liveness proof.** `CacheService` swallows Redis errors, so a
dead Valkey typically presents as *zeroed counters*, not as a thrown error — the snapshot then
reports `available: true` with a user who looks like they've used nothing today. This is the
same failure mode behind the "OTP expired for everyone" incidents. If quota numbers look
impossibly clean across multiple users, suspect Redis before you suspect the user.

**4. Streak comes from Neo4j, the least-trusted store in the stack.**
Known stale/duplicate `User` nodes exist from past manual Postgres deletions. A driver failure
or a missing node degrades to `available: false` + `unavailableReason` rather than failing the
whole snapshot. **`available: false` must never render as "0-day streak"** — it means "we
couldn't read it", which is a different sentence to a support agent.

**5. The mobile-emitted signals produce NOTHING until the next app release ships and users update.**
`client_error`, `paywall_viewed`, `upgrade_tapped`, `checkout_opened`, `checkout_abandoned`,
`purchase_failed`, `app_opened`, `app_backgrounded`, and `permissionGranted` on push tokens
are all emitted by app code that is **not in any released build at the time of writing**. The
backend accepts them today; nothing sends them yet. Those panels will be empty, and empty is
correct. The UI must distinguish **"no data yet — this ships with the next app release"** from
**"nothing happened"**. Same for `permissionGranted: null`, which means *the client build
predates the field* — it does **not** mean permission was denied.

**6. `api_usage` skips some traffic by design.**
Only **authenticated** requests are captured (`request.user.id` must exist), and the routes
`/sme/*`, `/health*`, `/config*` are skipped outright. Only the route **template** is stored
(`/pyq/:id/reveal`) — never ids, query strings, bodies, headers or IPs. The single exception
is `POST /quota/begin`, whose `feature` body field is stored **only** when it matches the
closed enum `chat_mentor | chat_sme | chat | flashcard | mnemonic | mains_eval | pyq_variation`.
So no PII can land in that column — and every AI feature that used to collapse into one
indistinguishable `/quota/begin` row is now labelled.

---

## 1. `GET /sme/users/:id/snapshot`

**Purpose:** the header of the user-detail page. One call, everything an agent needs in the
first five seconds. Every sub-read is bounded; a failing Redis or Neo4j degrades its own
field rather than the request.

**Query params**

| Param | Type | Default | Max | Meaning |
|---|---|---|---|---|
| `days` | int as string | `30` | `90` | Window for the `activity.counts` block only. Everything else (entitlement, quota, last-seen, clients) is current-state and ignores it. Non-numeric → default. |

**Response 200** (raw — this whole object is the body)

```json
{
  "user": {
    "id": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
    "supportCode": "1MQR-PGCT",
    "email": "aspirant@example.com",
    "name": "Meghana R.",
    "phoneNumber": "+919876543210",
    "phoneVerified": true,
    "status": "SUBSCRIBED",
    "provider": "cognito",
    "createdAt": "2026-03-02T06:11:04.000Z"
  },
  "entitlement": {
    "isPremium": true,
    "premiumState": "PREMIUM",
    "premiumExpiresAt": "2027-03-02T06:11:04.000Z",
    "subscriptionSource": "APPLE",
    "trialEndsAt": null,
    "trialDaysLeft": null,
    "recentOrders": [
      {
        "id": "ord_7f3a…",
        "status": "PAID",
        "amount": 4999,
        "currency": "INR",
        "planType": "ANNUAL",
        "paymentSource": "RAZORPAY",
        "premiumGrantedAt": "2026-06-16T11:02:41.000Z",
        "createdAt": "2026-06-16T11:02:10.000Z"
      }
    ]
  },
  "quota": {
    "available": true,
    "scope": "today_only",
    "istDate": "2026-07-23",
    "resetsAtISTMidnight": true,
    "premium": true,
    "features": { "chat_mentor": { "unlimited": true, "type": "daily" } },
    "unavailableReason": null
  },
  "streak": {
    "available": true,
    "currentStreak": 12,
    "maxStreak": 31,
    "lastActiveDate": "2026-07-23",
    "unavailableReason": null
  },
  "activity": {
    "windowDays": 30,
    "from": "2026-06-23T09:14:22.000Z",
    "to": "2026-07-23T09:14:22.000Z",
    "counts": {
      "auth_events": 4, "api_usage": 812, "orders": 1, "refunds": 0,
      "payment_events": 3, "user_question_attempts": 140,
      "simulation_attempts": 2, "user_document_progress": 9,
      "custom_tasks": 6, "user_content": 11,
      "psychometric_test_results": 1, "notification_history": 22,
      "sme_audit_log": 2, "feedback_reports": 1,
      "survey_responses": 0, "chat_message_feedback": 3
    },
    "total": 1017
  },
  "lastSeen": {
    "lastLoginAt": "2026-07-23T04:02:00.000Z",
    "lastActiveAt": "2026-07-23T08:55:12.000Z",
    "lastSessionAt": "2026-07-23T08:55:10.000Z",
    "lastRequestAt": "2026-07-23T08:55:12.000Z",
    "lastRequestRoute": "GET /tasks/today",
    "lastClientEventAt": "2026-07-23T04:01:58.000Z",
    "lastClientEventType": "login_success"
  },
  "clients": {
    "appVersions": [
      { "appVersion": "1.7.0", "platform": "ios", "lastSeenAt": "2026-07-23T08:55:12.000Z" }
    ],
    "distinctDeviceCount": 2,
    "deviceCountCapped": false,
    "pushTokens": [
      { "platform": "ios", "isActive": true, "permissionGranted": null, "lastUsedAt": "2026-07-22T18:30:00.000Z" }
    ]
  }
}
```

**Field meanings**

| Field | Meaning |
|---|---|
| `user.supportCode` | The code the user reads aloud (see §5). Derived, never stored. Empty string `""` if the id isn't a UUID — render nothing, not `""`. |
| `user.status` | Postgres lifecycle: `ACTIVE`, `SUBSCRIBED`, `INACTIVE`, `SUSPENDED`, `LOCKED`, `ONBOARDING`. **Never use `status === "ACTIVE"` as a premium signal** — that's the trial state. |
| `entitlement.isPremium` / `premiumState` | Computed by the shared `premium-check.util` (PostgreSQL `status` + `premiumExpiresAt` + `createdAt` + `trialEndsAt`). This is the authoritative entitlement answer. |
| `entitlement.trialEndsAt` / `trialDaysLeft` | Non-null **only** while the user is actually inside a trial (`status === 'ACTIVE'` and now < trial end). `trialDaysLeft` is ceil'd days. |
| `entitlement.recentOrders[]` | Last **5** orders, newest first. `amount` is **converted to major units** (rupees/dollars) — the DB stores paise/cents, this endpoint already divided by 100. Don't divide again. |
| `quota.*` | See limitation 3. `features` is `Record<featureKey, …>` whose value shape varies by cap type: `{ unlimited: true, type }` for premium; `{ used, limit, remaining, type: 'daily' \| 'lifetime' }`, `{ type: 'lifetime_per_subject', limit, perSubject: true }`, or `{ type: 'premium_only' }` for free. Render generically off `type`. |
| `streak.*` | See limitation 4. `lastActiveDate` is a Neo4j-supplied `YYYY-MM-DD` string. |
| `activity.retention` | Same object as the timeline's `meta.retention` (limitation 1) — the `api_usage` count is the one that goes quiet past 30 days. |
| `activity.counts` | One bounded `COUNT` per source over the window. Keys match the timeline `source` values, so a count tile can deep-link straight into `?sources=<key>`. **16 keys, not 17** — there is no `payments` count, because the `payments` table has no `user_id` and counting it would need the order fan-out. `custom_tasks` counts only user-created tasks (`sourceId IS NULL`) — auto-seeded planner content is excluded on purpose. `payment_events` counts on `receivedAt`; `simulation_attempts` on `startedAt`; everything else on `createdAt`. |
| `lastSeen.lastRequestAt/Route` | From `api_usage` — subject to limitation 1. `null` for a user who hasn't made an authenticated request since capture shipped / within retention. |
| `lastSeen.lastClientEventAt/Type` | Newest `auth_events` row of any type, **unbounded by the window** (the whole 90-day retention). |
| `clients.appVersions[]` | Distinct `(appVersion, platform)` pairs seen in `api_usage` within the window, newest first, capped at 20. Either field can be `null` for requests that didn't send the header. Multiple entries = the user upgraded, or is on two platforms. |
| `clients.distinctDeviceCount` | Distinct non-null `deviceId`s in `api_usage` within the window, probed up to **50**. `deviceCountCapped: true` means "50 or more" — show it as `50+`, never as exactly 50. |
| `clients.pushTokens[]` | Up to 20 device tokens, newest-updated first. `permissionGranted: null` = client build predates the field (limitation 5) — **not** "denied". `isActive: false` = token was invalidated by FCM/APNs. |

**Errors:** 404 if the user id doesn't exist. Quota and streak failures do **not** error — they
come back with `available: false`.

---

## 2. `GET /sme/users/:id/timeline`

**Purpose:** the merged, reverse-chronological record of everything that happened to this
user, across 17 tables. Each source is fetched with its own bounded single-table query and
merged in application code — there are no cross-table joins anywhere.

**Query params**

| Param | Type | Default | Cap | Meaning |
|---|---|---|---|---|
| `sources` | comma-separated | all 17 | — | Subset filter. See the source table below. Unknown value → **400** with the full valid list. Duplicates de-duped, order preserved. Empty/blank → all. |
| `from` | ISO datetime or `YYYY-MM-DD` | `to − days` | — | Inclusive lower bound. A bare `YYYY-MM-DD` is resolved as **IST** `00:00:00.000+05:30`, never UTC. |
| `to` | ISO datetime or `YYYY-MM-DD` | now | — | Inclusive upper bound. A bare `YYYY-MM-DD` → IST `23:59:59.999+05:30`. |
| `days` | int as string | `30` | `90` | Used only when `from` is omitted. |
| `limit` | int as string | `50` | `200`, then clamped | Rows per page. **Also** clamped so `sources.length × (limit + 1) ≤ 2000`. With all 17 sources the effective ceiling is **116**; ask for fewer sources to get a bigger page. `meta.limitClamped` tells you it happened. |
| `cursor` | opaque string | — | — | `nextCursor` from the previous page. Malformed → **400 `Malformed cursor.`** |

**Window is hard-capped at 90 days even when you pass both ends.** If `to − from > 90d`, `from`
is silently moved forward to `to − 90d`; `meta.from` reflects the window actually used, so
render `meta.from`/`meta.to`, not the values you sent. `from` after `to` → **400**.

**Response 200**

```json
{
  "items": [
    {
      "id": "api_usage:c1f0…",
      "rowId": "c1f0…",
      "source": "api_usage",
      "type": "POST /quota/begin [chat_mentor]",
      "at": "2026-07-23T08:55:12.000Z",
      "title": "POST /quota/begin [chat_mentor] → 200 (41ms)",
      "severity": "info",
      "data": {
        "method": "POST", "route": "/quota/begin", "status": 200,
        "durationMs": 41, "feature": "chat_mentor",
        "appVersion": "1.7.0", "platform": "ios", "deviceId": "6f2c…"
      }
    }
  ],
  "nextCursor": "MjAyNi0wNy0yM1QwODo1NToxMi4wMDBafGMxZjA",
  "hasMore": true,
  "meta": {
    "sources": ["auth_events", "api_usage", "…"],
    "limit": 50,
    "limitClamped": false,
    "rowsFetched": 634,
    "rowCap": 2000,
    "from": "2026-06-23T09:14:22.000Z",
    "to": "2026-07-23T09:14:22.000Z",
    "windowDays": 30,
    "retention": { "…": "see limitation 1" }
  }
}
```

**Item fields**

| Field | Meaning |
|---|---|
| `id` | `"<source>:<rowId>"`. Stable and unique across sources — use it as the React key. |
| `rowId` | The underlying table's primary key. Also the cursor tiebreaker. |
| `source` | One of the 17 canonical values below. Drives the icon/colour. |
| `type` | Sub-kind *within* the source — `login_failed`, `POST /quota/begin [chat_mentor]`, `PAID`, `correct`, `ISSUE/OPEN`, `razorpay.webhook`, … Not a closed enum; it is per-source and safe to display verbatim. |
| `at` | Event timestamp (UTC ISO). Whichever column that source is ordered by — see the table. |
| `title` | A pre-rendered, human-readable one-liner. **Use it.** It is built server-side per source, so the portal doesn't reimplement 17 formatters. |
| `severity` | `info` \| `warn` \| `error`. Drives colour only. `api_usage`: ≥500 → `error`, 400–499 → `warn`, else `info`. `auth_events`: `error` for the failure types listed in §3. Orders `FAILED` → `error`, `CANCELLED` → `warn`. Payments `FAILED` → `error`, `REFUNDED` → `warn`. Refunds always `warn`. Feedback `ISSUE` → `warn`, chat feedback `DOWN` → `warn`. |
| `data` | Source-specific structured payload for the expanded/detail view. Keys are listed per source below. Values may be `null`. |

**Sources and their `data` keys**

| `source` | Table | Ordered on | `data` keys |
|---|---|---|---|
| `auth_events` | `auth_events` | `createdAt` | `platform`, `appVersion`, `deviceId`, `deviceModel`, `osVersion`, `errorCode`, `message`, `metadata`, `identifierMasked`, `ip` |
| `api_usage` | `api_usage` | `createdAt` | `method`, `route`, `status`, `durationMs`, `feature`, `appVersion`, `platform`, `deviceId` |
| `orders` | `orders` | `createdAt` | `status`, `amount` (major units), `currency`, `planType`, `paymentSource`, `premiumGrantedAt`, `razorpayOrderId`, `appleTransactionId` |
| `payments` | `payments` | `createdAt` | `orderId`, `status`, `amount` (major units), `currency`, `method`, `errorCode`, `errorReason`, `capturedAt`, `razorpayPaymentId`, `appleTransactionId` |
| `payment_events` | `payment_events` | `receivedAt` | `provider`, `kind`, `type`, `result`, `error`, `externalId`, `orderId`, `processedAt` |
| `refunds` | `refunds` | `createdAt` | `provider`, `status`, `amount` (major units, nullable), `currency`, `reason`, `paymentId`, `providerRefundId` |
| `user_question_attempts` | `user_question_attempts` | `createdAt` | `documentId`, `questionId`, `selectedOption`, `isCorrect` |
| `simulation_attempts` | `simulation_attempts` | **`startedAt`** | `simulationId`, `submittedAt`, `isAutoSubmitted`, `correctCount`, `wrongCount`, `skippedCount`, `rawScore`, `accuracy`, `timeTakenSeconds` |
| `user_document_progress` | `user_document_progress` | `createdAt` | `documentId`, `questionsAnswered`, `questionsCorrect`, `totalQuestions`, `isCompleted`, `completedAt` |
| `custom_tasks` | `custom_tasks` | `createdAt` | `tag`, `scheduledDate`, `scheduledTime`, `isCompleted`, `completedAt` — **only user-created tasks** (`sourceId IS NULL`) |
| `user_content` | `user_content` | `createdAt` | `type` (the AI generation kind) |
| `psychometric_test_results` | `psychometric_test_results` | `createdAt` | `testSetId`, `totalScore`, `totalQuestions`, `correctAnswers`, `skippedAnswers`, `durationMs`, `isAutoSubmitted` |
| `notification_history` | `notification_history` | `createdAt` | `title`, `body`, `type`, `entityId`, `isRead` |
| `sme_audit_log` | `sme_audit_log` | `createdAt` | `action`, `actorNote`, `before`, `after` — includes `SUPPORT_NOTE` rows from §4 |
| `feedback_reports` | `feedback_reports` | `createdAt` | `type`, `status`, `categoryKey`, `text`, `contextType`, `contextId`, `platform`, `appVersion`, `userTier` |
| `survey_responses` | `survey_responses` | `createdAt` | `surveyId`, `status`, `answeredAt`, `snoozeCount`, `snoozedUntil`, `userTier`, `platform` |
| `chat_message_feedback` | `chat_message_feedback` | `createdAt` | `messageId`, `mode`, `rating`, `subject`, `chips`, `text`, `difyForwarded`, `difyError` |

**`sources` aliases** — accepted on input, always **normalised to the canonical name** in
`meta.sources` and in every item's `source`. Match on the canonical value in your code.

| You may send | Resolves to | Why |
|---|---|---|
| `webhook_events` | `payment_events` | The `webhook_events` table has **no `user_id` column** — it's a raw provider-event dedup ledger keyed on `eventId` and cannot be filtered per user. `payment_events` is the per-user view of the same provider traffic (`provider`, `kind`, `type`, `result`, `userId`, `orderId`) with an indexed `user_id`. |
| `webhooks` | `payment_events` | shorthand |
| `notes` | `sme_audit_log` | support notes live in the audit log |
| `staff` | `sme_audit_log` | shorthand |

Input is lowercased and trimmed before matching. Anything else → `400` listing all valid
sources and aliases.

**Pagination.** Opaque keyset cursor — `base64url("<ISO timestamp>|<rowId>")`. Treat it as
opaque; the encoding is an implementation detail. Loop `while (page.hasMore)` passing
`page.nextCursor` and *the identical* `sources`/`from`/`to`/`days`/`limit` params. Changing
the window or source set mid-scroll invalidates the position. `nextCursor` is `null` iff
`hasMore` is `false`.

**`meta` fields**

| Field | Meaning |
|---|---|
| `sources` | Canonical sources actually queried, after alias resolution and de-duplication. |
| `limit` | Effective page size after both clamps. |
| `limitClamped` | `true` when your requested `limit` was reduced by the row-cap rule. Worth a quiet note in the UI when someone asks for 200 rows across all sources. |
| `rowsFetched` | Rows materialised out of Postgres for this page, across all sources (≥ `items.length`, since each source over-fetches by one to detect `hasMore`). Diagnostic only. |
| `rowCap` | Always `2000`. The absolute per-page materialisation ceiling. |
| `from` / `to` / `windowDays` | The window **actually used** after the 90-day clamp. Render these. |
| `retention` | What the window can and cannot contain — see limitation 1. Render `note` when it is non-null. |

**Errors:** 400 (bad source / malformed cursor / `from` after `to` / unparseable date), 404
(unknown user).

---

## 3. `GET /sme/users/:id/incidents`

**Purpose:** the "what went wrong" shortcut. Purely **derived** from data already captured —
no new tracking. Failures are pulled from two sources, merged newest-first, and clustered by
time proximity so an agent reads *"at 14:22 IST, 3 failures on POST /payments/verify"* instead
of scrolling a timeline.

**What counts as a failure**
- `api_usage` rows with `status >= 400` (both 4xx and 5xx).
- `auth_events` rows whose `eventType` is one of:
  `client_error`, `login_failed`, `otp_send_failed`, `otp_verify_failed`, `refresh_failed`,
  `purchase_failed`, `forced_logout`.

**Query params**

| Param | Type | Default | Max | Meaning |
|---|---|---|---|---|
| `from` / `to` / `days` | — | `days=30` | 90 days | Identical semantics to the timeline, including IST date parsing and the hard 90-day span clamp. |
| `limit` | int as string | `20` | `50` | Incidents (clusters) per page. |
| `gapMinutes` | int as string | `5` | `60` | Max quiet gap between consecutive failures inside one incident. Widen it to merge a flappy session into one story; narrow it to split. |
| `cursor` | opaque | — | — | `nextCursor` from the previous page. |

Up to **250 rows per source / 500 total** are scanned per page before clustering.

**Response 200**

```json
{
  "incidents": [
    {
      "id": "incident:9c2b…",
      "startedAt": "2026-07-23T08:52:01.000Z",
      "endedAt": "2026-07-23T08:54:40.000Z",
      "startedAtIST": "14:22",
      "istDate": "2026-07-23",
      "failureCount": 3,
      "severity": "error",
      "title": "at 14:22 IST, 3 failures on POST /payments/verify (+1 other)",
      "sources": ["api_usage", "auth_events"],
      "byLabel": [
        { "label": "POST /payments/verify", "count": 2 },
        { "label": "purchase_failed", "count": 1 }
      ],
      "byStatus": [{ "status": 500, "count": 2 }],
      "samples": [ { "…": "SmeTimelineItem, identical shape to §2" } ]
    }
  ],
  "nextCursor": "…",
  "hasMore": false,
  "meta": {
    "limit": 20,
    "gapMinutes": 5,
    "rowsScanned": 87,
    "rowCap": 500,
    "perSourceRowCap": 250,
    "rowCapHit": false,
    "truncatedSources": [],
    "from": "2026-06-23T09:14:22.000Z",
    "to": "2026-07-23T09:14:22.000Z",
    "windowDays": 30,
    "retention": { "…": "see limitation 1" }
  }
}
```

| Field | Meaning |
|---|---|
| `id` | `"incident:<rowId of the oldest failure>"`. Stable for a fixed window + `gapMinutes`; **changes if `gapMinutes` changes** (different clustering). Don't persist it. |
| `oldestRowId` | Primary key of the cluster's oldest row — the same one the id embeds, and what the pagination cursor is built from. Do **not** read this off `samples`: that array is truncated to 5, so on a longer incident its last element is the 5th-newest row, not the oldest. |
| `startedAt` / `endedAt` | Oldest and newest failure in the cluster. A single-row incident has both equal. |
| `startedAtIST` | `"HH:MM"` in IST — what the agent says out loud to the user. |
| `istDate` | IST calendar date of `startedAt`. Group headers should use this, not the UTC date. |
| `failureCount` | Rows in the cluster (may exceed `samples.length`). |
| `severity` | `error` if any member row is `error`, otherwise `warn`. Never `info` — everything here is a failure. |
| `title` | Pre-rendered summary sentence. Use it as the row headline. |
| `sources` | Distinct sources contributing, in first-seen order. |
| `byLabel[]` | Failure labels tallied, **descending by count** then alphabetical. `label` is the `type` of the underlying item (`METHOD /route` or the event type). This is the "what broke" breakdown. |
| `byStatus[]` | HTTP statuses tallied the same way. Empty when the cluster is entirely `auth_events` (those have no status). |
| `samples[]` | Up to **5** member rows, newest-first, in the exact `SmeTimelineItem` shape from §2 — so one component renders both screens. |

**Pagination caveat, stated plainly:** clustering happens **within a page**. An incident that
straddles a page boundary is **split at the boundary** — you may see the tail of a burst at the
bottom of page 1 and its head at the top of page 2. If that matters, raise `limit` rather than
stitching clusters client-side. `nextCursor` points at the **true oldest row** of the last
incident on the page (`oldestRowId`), so an incident longer than its 5-row `samples` array is
never re-served on the following page with a partial `failureCount`.

**`meta.rowCapHit` / `truncatedSources`** — the 500-row budget is split **per source**
(`perSourceRowCap`, currently 250 each for `api_usage` and `auth_events`), and each is capped
independently. `rowCapHit` is `true` when **any one** source was truncated — which happens at
250 rows, long before `rowsScanned` reaches 500 — and `truncatedSources` names them.
`hasMore` alone will not tell you this. Surface it ("showing the most recent 250 request
failures") — a user in this state is having a very bad day and the agent should know the list
is truncated.

---

## 4. `POST /sme/users/:id/notes`

**Purpose:** a staff note on a user. Stored as an `sme_audit_log` row with
`action = 'SUPPORT_NOTE'` — no new table — and it surfaces on the timeline under source
`sme_audit_log` (also reachable via `?sources=notes`).

**Request**

```json
{ "note": "Called back — Razorpay double-charge, refund raised #RF2291", "category": "refund", "author": "priya@stanzasoft.com" }
```

| Field | Required | Constraint | Stored as |
|---|---|---|---|
| `note` | yes | string, 1–2000 chars | `sme_audit_log.actor_note` |
| `category` | no | string, ≤60 chars | inside `sme_audit_log.after` |
| `author` | no | string, ≤120 chars | inside `sme_audit_log.after` |

This body **is** validated by the global `ValidationPipe` (unlike the diagnostics sink) — a
missing or over-long `note` returns `400` with the class-validator messages in `message`.

⚠️ **`author` is self-reported and unverified.** The SME surface authenticates with a *shared*
`x-api-key` that carries no identity, so the server cannot know who wrote the note. If the
portal has its own logged-in user, pass it here — otherwise the note is anonymous. Do not
present `author` as an attested identity in the UI.

**Response 201** (raw)

```json
{
  "id": "a13c…",
  "action": "SUPPORT_NOTE",
  "targetUserId": "0d2f8b41-…",
  "note": "Called back — Razorpay double-charge, refund raised #RF2291",
  "category": "refund",
  "author": "priya@stanzasoft.com",
  "createdAt": "2026-07-23T09:20:44.000Z"
}
```

Unlike every other SME audit write (which is deliberately best-effort and swallows failures),
this one is written directly and **a DB failure surfaces as a 5xx** — because here the note *is*
the entire request, and silently losing it would be worse than an error. Retry on failure; the
optimistic row in the UI must be rolled back, not left on screen.

**Errors:** 400 (validation), 404 (unknown user).

---

## 5. `GET /sme/users/by-support-code/:code`

**Purpose:** turn the short code a user reads out on a phone call into an account, without
asking them to spell an email address.

**Response 200**

```json
{
  "id": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
  "supportCode": "1MQR-PGCT",
  "email": "aspirant@example.com",
  "name": "Meghana R.",
  "phoneNumber": "+919876543210",
  "status": "SUBSCRIBED",
  "isPremium": true,
  "premiumState": "PREMIUM",
  "createdAt": "2026-03-02T06:11:04.000Z"
}
```

Feed `id` straight into §1–§4.

**Errors:** `400` unusable code (see decode rules), `404` `No user matches support code …`,
`409` ambiguous (candidate ids are **not** returned — see the filter limitation at the top).

### The support-code algorithm — exact spec

Deterministic, derived purely from `UserAuth.id`. **Nothing is stored**; there is no support-code
column and no lookup table. The mobile app derives the same code on-device from the same id,
and the server decodes it back to an `id LIKE 'prefix%'` lookup. Mirror this exactly.

**ENCODE(userId) → `"XXXX-XXXX"`**

1. Take the user id (a UUID v4), lowercase it, remove every `-` → 32 hex characters.
2. Take the **first 10 hex characters** → a 40-bit unsigned integer.
3. Encode that integer as **exactly 8 symbols** of Crockford Base32 (40 bits ÷ 5 bits/symbol),
   most-significant symbol first, over the alphabet:
   ```
   0123456789ABCDEFGHJKMNPQRSTVWXYZ
   ```
   The alphabet deliberately **excludes `I`, `L`, `O` and `U`** — so the code can't be misheard
   ("eye"/"one", "oh"/"zero") and can't spell an obscenity.
4. Render as two hyphen-separated groups of 4: `K3M9-7TQ2`.

Worked example: id `0d2f8b41-9a3c-…` → hex prefix `0d2f8b419a` → `1MQR-PGCT`.

**DECODE(code) → user-id prefix — deliberately lenient, because it arrives over a phone call**

1. Uppercase the input; **drop every character outside `[0-9A-Z]`** — hyphens, spaces, dots,
   underscores and punctuation all vanish. ⚠️ Note what this does **not** do: it does not strip
   letters. A typed prefix like `PM-K3M97TQ2` keeps its `P` and `M`, becomes 10 symbols, and is
   rejected in step 3. Only separators are forgiving, not extra words.
2. Apply the Crockford read-aloud aliases: **`O` → `0`**, **`I` → `1`**, **`L` → `1`**.
3. Require **exactly 8 symbols**, all members of the alphabet. Otherwise `400`
   (`Support code must be 8 symbols (e.g. K3M9-7TQ2) after removing separators.` or
   `Support code contains an unusable symbol 'X'. Valid symbols: …`).
4. Decode to the 40-bit integer; render back as **10 lowercase hex characters**, zero-padded.
5. Match users whose `id` starts with `hex[0..8] + "-" + hex[8..10]` (a UUID always has a `-` at
   index 8) — i.e. `WHERE id LIKE 'prefix%'`, capped at 5 rows.
   ⚠️ Being precise about the cost: `LIKE 'prefix%'` only uses a btree index under
   `text_pattern_ops` or a `C` collation, and neither is declared on `user_auth`, so under the
   default collation Postgres **seq-scans** this. At a user base in the thousands that is
   sub-millisecond and it is left alone deliberately — but do not cite it as an indexed lookup.

**Portal implications**

- **Display:** always show the hyphenated, uppercase form (`K3M9-7TQ2`) — that's what the app
  shows and what the user is reading.
- **Input:** be permissive about separators and case. `K3M9-7TQ2`, `k3m9 7tq2`, `k3m97tq2` and
  `K3M9.7TQ2` all resolve identically, and `K3M9-7TQ0` typed as `K3M9-7TQO` (letter O) resolves
  to the same user thanks to the aliases. Do **not** add a strict input mask — it will reject
  codes the server would have happily accepted. Do trim/uppercase for display only.
- **Don't index or cache on the code.** It is a function of the id — derive it, don't store it.
- **Collisions:** 2^40 ≈ 1.1×10¹² over a user base in the thousands. The 409 path exists so the
  server never silently returns the wrong human, not because it's expected.

---

## Appendix A — the client-event sink (`POST /diagnostics/events`)

Not an SME endpoint, but it is where half the timeline's `auth_events` rows come from, so the
portal team should know how they arrive.

- **Routes:** `POST /api/v1/diagnostics/events` **and** `POST /api/v1/diagnostics/auth-events` —
  two paths, **one handler, identical semantics**. `events` is the honest name now that the sink
  carries general client events; `auth-events` is kept **forever** because 1.6/1.7 builds already
  in the field call it and will never be updated.
- **`@Public()` by design** — the whole point is capturing failures *before* a user is
  authenticated (login/OTP/refresh). Not an `x-api-key` surface.
- **Responses:** `204` accepted (no body), `429` rate-limited (**silent — no body, nothing
  stored, not an error**), `400` only for an unknown `eventType`.
- **Rate limits:** 60/hour per `deviceId` (falling back to IP), plus a secondary 300/hour per IP
  alone so rotating device ids can't buy a fresh bucket.
- **Validation stance:** `eventType` is the **only** hard gate. Every other field is coerced or
  tolerated — a diagnostics sink that 400s on a stray field is worse than a lossy one. A DB
  insert failure is swallowed and logged; the client still gets `204`.
- **Privacy:** the raw `identifier` (email/phone) is **never persisted**. The server normalises
  it (email → lowercased; phone → digits only), stores a last-4 mask (`******7935`) plus a
  SHA-256 hex hash, and drops the original. That's why the timeline shows `identifierMasked`
  and never a readable identifier.
- **`metadata`:** flat object only — max **20 keys**, keys trimmed to 64 chars, values must be
  string / finite number / boolean (strings trimmed to 200 chars). Arrays, nested objects,
  `null`, `NaN`, class instances and `Object.create(null)` are **dropped**, not flattened.
  Nothing survives → the column is SQL `NULL`.

**Accepted `eventType` values** (strict allow-list, but stored as a plain `String` column — new
types need a constant edit, never a migration):

| Group | Types | Live today? |
|---|---|---|
| Auth lifecycle | `login_attempt`, `login_success`, `login_failed`, `otp_send_failed`, `otp_verify_failed`, `refresh_failed`, `forced_logout` | **Yes** — emitted by shipped builds |
| Client errors | `client_error` | No — next app release |
| Purchase funnel | `paywall_viewed`, `upgrade_tapped`, `checkout_opened`, `checkout_abandoned`, `purchase_failed` | No — next app release |
| App lifecycle | `app_opened`, `app_backgrounded` | No — next app release |
| Chat pointer | `chat_conversation_started` | **Yes** — but written **server-side** by `POST /quota/finalize`, not by the sink, and only when the app sends a `conversationId` (also next release). `metadata = { feature, conversationId }`, conversationId truncated to 128 chars. |

---

## Appendix B — schema changes behind this surface

Migration `prisma/migrations/20260723170000_activity_trail_capture/migration.sql`. All additive
and nullable; hand-applied (the `_prisma_migrations` table is not a reliable source of truth on
this project — verify via `information_schema`, not Prisma's migration state).

| Change | Why it matters to the portal |
|---|---|
| `api_usage.feature TEXT` | Sub-route discriminator for `/quota/begin` only. **Populated from the deploy date onward** — older rows are `null` even for chat calls. A `null` feature on a `/quota/begin` row means "before this shipped", not "unknown feature". |
| `auth_events.metadata JSON` | The per-event payload. `null` on every row written before the deploy. |
| `device_tokens.permission_granted BOOLEAN` | `null` until the next app release. **Never render `null` as denied.** |
| `api_usage(device_id, created_at)` index | Makes device/account-sharing queries viable. |
| `api_usage(feature, created_at)` index | Feature-level usage queries. |
| `psychometric_test_results(userId, createdAt)` index | This table previously declared **no indexes at all**, so every per-user read seq-scanned it. |

---

## Hard reminders

- **Success bodies are raw. No `{ success, data }`. Branch on HTTP status.**
- Render `meta.from` / `meta.to` — not the window you asked for. The server clamps to 90 days.
- `available: false` on quota/streak means "couldn't read it", **not** zero.
- `permissionGranted: null` means "old client build", **not** denied.
- `deviceCountCapped: true` means `50+`, not `50`.
- Money fields on this surface are already in major units — don't divide by 100 again.
- Timestamps are UTC ISO; IST-labelled fields (`istDate`, `startedAtIST`) are already converted.
  Group by `istDate`, not by the UTC date of `at`.
- `api_usage` is a rolling ~30 days and starts at the interceptor's deploy date. Check
  `min(created_at)` before promising anything — and render `retention.note` when it is set.
- `incidents[].samples` is capped at 5. Paginate on `nextCursor` / `oldestRowId`, never on the
  last element of `samples`.
- The timeline proves a chat happened and gives you its Dify id. It does not show what was said.
