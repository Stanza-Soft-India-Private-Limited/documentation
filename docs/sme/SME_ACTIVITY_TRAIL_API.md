# SME — User Activity Trail API

> Changed 2026-09-21 — stale-text pass (telemetry live in 2.0, version ladder, links). Exam-dimension changes follow in [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md).
> Changed 2026-09-21 — exam dimension on users, orders, offers, banners; two BREAKING calls (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

> ## ⛔ BEFORE YOU WRITE ANY UI CODE — INVOKE THE `frontend-design` SKILL
> If you are a Claude/coding session building anything on top of these endpoints, your
> **first action** is to invoke the **`frontend-design`** skill (`/frontend-design`). Do
> this **even if no one asked you to** and even if the surrounding task didn't mention
> design. Every panel, timeline row, incident cluster and empty state below must be
> designed through that skill — do not reach for a default admin-template layout. This is
> a hard requirement, not a suggestion.
>
> Design for what this contract actually becomes: **the support desk's answer sheet.** An
> internal operator has a PrepMonkey user on the phone or in a WhatsApp thread *right now*
> saying something went wrong, and is reading this screen while the user waits. It has to
> make the operator competent in about ten seconds — who this is, whether they're paid,
> whether something is broken — and only then let them dig. Audience = internal support/ops
> staff under time pressure, not analysts and not consumers. The job of the screen is *fast
> situational awareness*, then *drill-down*. Anything that costs the operator a second of
> reading while a human is on the line is a design failure.
>
> The information architecture that goes with this contract is a separate document —
> [`SME_USER_DETAIL_PAGE_GUIDE.md`](./SME_USER_DETAIL_PAGE_GUIDE.md). Read both.
>
> Paste this with the docs when you brief an agent:
>
> ```
> Invoke the /frontend-design skill first, before writing any UI code.
> Then build the SME user-detail page per SME_USER_DETAIL_PAGE_GUIDE.md,
> using SME_ACTIVITY_TRAIL_API.md as the authoritative field contract.
> This is a support-desk screen read under time pressure with a user on the phone:
> optimise for ten-second situational awareness first, drill-down second.
> Responses are RAW JSON — there is no {success,data} envelope; branch on HTTP status.
> Render every "unavailable"/"not recorded yet" state explicitly; never show 0 for it.
> Use the server-rendered `title` strings; do not reimplement 17 per-source formatters.
> ```

The support/debug surface that answers **"what happened to this user, end to end"**.
Five endpoints under `/sme/users/*`: a snapshot header, a merged timeline, auto-grouped
incidents, staff notes, and support-code lookup.

**Status: all five are deployed and were exercised against a real production account on
2026-07-23** — snapshot, timeline (17 sources, cursor-paginated), incidents, and
support-code lookup all returned `200` with the `x-api-key`. `POST …/notes` is live on the
same controller and guard; it is the only write, so it was not fired against production.

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
> last week. Do not port an envelope-unwrapping helper from another integration. The
> pre-2026-07-23 copy of [`SME_FEEDBACK_API.md`](./SME_FEEDBACK_API.md) claimed "all
> responses ride the global envelope"; that copy is superseded and the current one carries
> the correction at the top. If any doc anywhere disagrees with this paragraph, the code is
> the authority.
>
> ⚠️ Not the same thing: a few endpoints hand-roll a **pagination** wrapper inside their own
> controller — `GET /sme/users` and the feedback lists return
> `{ data: [...], total, page, limit, hasMore }`. That is a documented per-endpoint shape,
> not a global envelope, and it does not appear on any endpoint in this document. Trust the
> shape printed under each endpoint; never a global rule.

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

**5. The mobile-emitted signals are LIVE as of app 2.0, but only from devices that have updated.**
`client_error`, `paywall_viewed`, `upgrade_tapped`, `checkout_opened`, `checkout_abandoned`,
`purchase_failed`, `chat_conversation_started`, `app_opened`, `app_backgrounded`, and
`permissionGranted` on push tokens are all emitted by **app 2.0, released to both stores on
2026-09-17** (Android 27, iOS 2.0 build 3). Older builds emit none of them and never will, so
volume on these panels grows with adoption rather than arriving all at once. A thin panel is
therefore a *coverage* statement, not a "nothing happened" statement, and the UI must still
distinguish **"this user is on an older build"** from **"nothing happened"**. Same for
`permissionGranted: null`, which means *the client build predates the field* — it does **not**
mean permission was denied.

**6. `api_usage` skips some traffic by design.**
Only **authenticated** requests are captured (`request.user.id` must exist), and the routes
`/sme/*`, `/health*`, `/config*` are skipped outright. Only the route **template** is stored
(`/api/v1/pyq/:id/reveal`) — never ids, query strings, bodies, headers or IPs.

⚠️ **The stored route keeps the `/api/v1` global prefix.** It is Express's
`request.route.path` verbatim, so a real row reads `/api/v1/tasks/today`, not
`/tasks/today` — verified live on 2026-07-23. The skip-list match strips the prefix before
comparing, but the **stored value does not**. If you group, filter or pretty-print routes in
the portal, strip `^/api/v[0-9]+` yourself; do not assume the leading segment is the module.
(The handful of handlers Nest excludes from the prefix — `config` — genuinely store the bare
path, so handle both forms.) The single exception
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

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/users/0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30/snapshot` — verbatim, nothing
elided:

```json
{
  "user": {
    "id": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
    "supportCode": "1MQR-PGCT",
    "email": "aspirant@example.com",
    "name": "A. Sharma",
    "phoneNumber": "+91XXXXXXXXXX",
    "phoneVerified": true,
    "status": "SUBSCRIBED",
    "provider": "google",
    "createdAt": "2026-09-09T11:49:43.440Z"
  },
  "entitlement": {
    "activeExamId": "appsc-group-1",
    "entitlements": [
      {
        "examId": "appsc-group-1",
        "accessTier": "paid",
        "status": "LIVE",
        "source": "RAZORPAY",
        "expiresAt": "2026-10-16T12:09:32.616Z",
        "isTrial": false,
        "trialEndsAt": null
      }
    ],
    "isPremium": true,
    "premiumState": "Premium",
    "premiumExpiresAt": "2026-10-16T12:09:32.616Z",
    "subscriptionSource": "RAZORPAY",
    "trialEndsAt": null,
    "trialDaysLeft": null,
    "recentOrders": [
      {
        "id": "5b71d908-3c42-4f6b-a087-13e9d5c82b64",
        "status": "PAID",
        "amount": 499,
        "currency": "INR",
        "planType": "MONTHLY",
        "paymentSource": "RAZORPAY",
        "examId": "appsc-group-1",
        "premiumGrantedAt": "2026-09-16T12:09:32.623Z",
        "createdAt": "2026-09-16T12:09:32.607Z"
      }
    ]
  },
  "quota": {
    "available": true,
    "scope": "today_only",
    "examId": "appsc-group-1",
    "istDate": "2026-09-21",
    "resetsAtISTMidnight": true,
    "premium": true,
    "features": {
      "chat_mentor":      { "unlimited": true, "type": "daily", "resetsAt": "2026-09-21T18:30:00.000Z" },
      "chat_sme":         { "unlimited": true, "type": "daily", "resetsAt": "2026-09-21T18:30:00.000Z" },
      "chat":             { "unlimited": true, "type": "daily", "resetsAt": "2026-09-21T18:30:00.000Z" },
      "flashcard":        { "unlimited": true, "type": "daily", "resetsAt": "2026-09-21T18:30:00.000Z" },
      "mnemonic":         { "unlimited": true, "type": "daily", "resetsAt": "2026-09-21T18:30:00.000Z" },
      "mains_eval":       { "unlimited": true, "type": "daily", "resetsAt": "2026-09-21T18:30:00.000Z" },
      "pyq_reveal":       { "unlimited": true, "type": "daily", "resetsAt": "2026-09-21T18:30:00.000Z" },
      "pyq_mains_reveal": { "unlimited": true, "type": "daily", "resetsAt": "2026-09-21T18:30:00.000Z" },
      "content_doc":      { "unlimited": true, "type": "lifetime_per_subject" },
      "pyq_variation":    { "unlimited": true, "type": "premium_only" }
    },
    "unavailableReason": null
  },
  "streak": {
    "available": false,
    "currentStreak": null,
    "maxStreak": null,
    "lastActiveDate": null,
    "unavailableReason": "Neo4j is unavailable (driver not initialised)"
  },
  "activity": {
    "windowDays": 30,
    "from": "2026-08-22T14:37:53.831Z",
    "to": "2026-09-21T14:37:53.831Z",
    "counts": {
      "auth_events": 116, "api_usage": 3528, "orders": 1, "refunds": 0,
      "payment_events": 0, "user_question_attempts": 0,
      "simulation_attempts": 1, "user_document_progress": 0,
      "custom_tasks": 0, "user_content": 2,
      "psychometric_test_results": 0, "notification_history": 3,
      "sme_audit_log": 0, "feedback_reports": 0,
      "survey_responses": 0, "chat_message_feedback": 0
    },
    "total": 3651,
    "retention": {
      "apiUsageRawRetentionDays": 30,
      "apiUsageCompleteFrom": "2026-08-22T14:37:53.831Z",
      "windowExceedsApiUsageRetention": false,
      "note": null
    }
  },
  "lastSeen": {
    "lastLoginAt": "2026-09-16T16:29:40.735Z",
    "lastActiveAt": null,
    "lastSessionAt": "2026-09-21T14:32:47.904Z",
    "lastRequestAt": "2026-09-21T14:32:49.947Z",
    "lastRequestRoute": "GET /api/v1/pyq/weak-topics",
    "lastClientEventAt": "2026-09-16T17:15:30.056Z",
    "lastClientEventType": "app_opened"
  },
  "clients": {
    "appVersions": [
      { "appVersion": null,   "platform": null,      "lastSeenAt": "2026-09-21T14:32:49.947Z" },
      { "appVersion": "2.0",  "platform": "ios",     "lastSeenAt": "2026-09-16T17:15:32.500Z" },
      { "appVersion": "2.0",  "platform": "android", "lastSeenAt": "2026-09-16T15:10:25.668Z" },
      { "appVersion": "1.10", "platform": "android", "lastSeenAt": "2026-09-16T14:53:56.280Z" },
      { "appVersion": "1.9",  "platform": "android", "lastSeenAt": "2026-09-16T14:50:46.264Z" },
      { "appVersion": "1.9",  "platform": "ios",     "lastSeenAt": "2026-09-10T08:34:29.894Z" }
    ],
    "distinctDeviceCount": 3,
    "deviceCountCapped": false,
    "pushTokens": [
      { "platform": "ios",     "isActive": true, "permissionGranted": true, "lastUsedAt": "2026-09-16T16:29:46.000Z" },
      { "platform": "android", "isActive": true, "permissionGranted": true, "lastUsedAt": "2026-09-16T15:10:19.334Z" }
    ]
  }
}
```

**Three things in that capture the portal must handle, and they are easy to miss in a
hand-written mock:**

1. **`streak.available: false` with a populated `unavailableReason`.** Staging's Neo4j is
   unreachable, and this is exactly the degraded shape the endpoint is designed to
   return: `currentStreak` / `maxStreak` / `lastActiveDate` all `null`, and a string
   saying why. **Render the reason, not a zero.** "Streak unavailable" and "streak is 0"
   are different answers to a support ticket. The same contract holds in production for
   any transient Neo4j outage.
2. **`clients.appVersions[0]` has `appVersion: null` and `platform: null`.** That is the
   bucket for requests that arrived without version headers (server-side / older
   clients). It sorts first because it is the most recent `lastSeenAt`. Filter nulls out
   of a "which app version are they on?" readout, or the answer is blank.
3. **`lastActiveAt` is `null` while `lastRequestAt` is minutes old.** They come from
   different sources (see the field table) and do not degrade together.

**Field meanings**

| Field | Meaning |
|---|---|
| `user.supportCode` | The code the user reads aloud (see §5). Derived, never stored. Empty string `""` if the id isn't a UUID — render nothing, not `""`. |
| `user.status` | Postgres lifecycle: `ACTIVE`, `SUBSCRIBED`, `INACTIVE`, `SUSPENDED`, `LOCKED`, `ONBOARDING`. **Never use `status === "ACTIVE"` as a premium signal** — that's the trial state. |
| `entitlement.activeExamId` | **Who this person is** — `user_profiles.active_exam_id`. **Nullable:** `null` means they have never picked an exam, and it is deliberately *not* coerced to `upsc-cse`, because "never chose" and "chose UPSC" are different facts and the support agent is the one who needs to tell them apart. **Never answer "which exams did they buy?" from this field.** |
| `entitlement.entitlements[]` | **What this person holds** — one entry per `user_exam_entitlements` row, from the same builder that powers `GET /sme/users` and `GET /sme/users/:id`, so a list row and this header can never disagree. Fields: `examId`, `accessTier` (`free` \| `paid`, the exam's tier at read time), `status` (`LIVE` \| `EXPIRED` \| `REVOKED` — **REVOKED beats an expiry still in the future**, which is exactly what a refund or an SME revoke means), `source` (`APPLE` \| `RAZORPAY` \| `MANUAL`, **per row** — not the person-level `subscriptionSource`), `expiresAt` (`null` = perpetual, not unknown), `isTrial`, `trialEndsAt`. **Empty array for a trial-only user** — a trial creates no row, and `premiumState`/`trialEndsAt` are what describe them. |
| `entitlement.entitlements[].isTrial` | "The row has stopped granting anything, but the person-level trial still covers this exam" — evaluated with **this exam's** trial length. Never true alongside `status: "LIVE"`, or every live subscriber would also read as on trial. |
| `entitlement.isPremium` / `premiumState` | Computed by the shared `premium-check.util` (PostgreSQL `status` + `premiumExpiresAt` + `createdAt` + `trialEndsAt`). This is the authoritative entitlement answer. **`premiumState` is a closed set of five human-readable labels, title-cased, one with a space: `Premium` · `Trial` · `Trial Ended` · `Churned` · `Downloaded`.** It is *not* SCREAMING_CASE and it is *not* the same vocabulary as the Wylto CRM statuses — the two were deliberately decoupled so a marketing rename cannot change this response. Match on the exact strings above. |
| `entitlement.trialEndsAt` / `trialDaysLeft` | Non-null **only** while the user is actually inside a trial (`status === 'ACTIVE'` and now < trial end). `trialDaysLeft` is ceil'd days. |
| `entitlement.recentOrders[]` | Last **5** orders, newest first. `amount` is **converted to major units** (rupees/dollars) — the DB stores paise/cents, this endpoint already divided by 100. Don't divide again. |
| `entitlement.recentOrders[].examId` | Which exam the money bought. **Nullable** — Apple's payloads carry no exam, so an order whose signed product matched no `exam_plans` row is recorded unresolved rather than guessed. Those are the `UNRESOLVED_EXAM` reconcile cases (`SME_PORTAL_API.md` §2.3). |
| `quota.examId` | **Which exam these quota numbers are for** — the user's `activeExamId`, or `upsc-cse` when unset. Caps come from that exam's `feature_caps` and the exemption from that exam's entitlement, so an unlabelled quota panel is a number nobody can act on. Present **even when `available: false`** — "which exam did we fail to read" is part of the diagnosis. Before 2026-09-21 this panel always reported UPSC's allowance, whoever you were looking at. |
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

### How to use this data

**Questions it answers.** "Who am I talking to?" · "Are they actually paying, and through
which store?" · "Have they hit today's cap?" · "Which build are they on?" · "Are they even
reachable by push?" All of it in one round-trip, before the operator has finished saying hello.

**The decision it drives.** Whether this is a *billing* conversation, a *quota* conversation,
a *stale-build* conversation, or a genuine bug — which is the only branch that costs
engineering time. Three fields settle it: `entitlement.premiumState`,
`quota.features[…].remaining`, and `clients.appVersions[0].appVersion`.

**What a good visualisation is.** A single header band, not a grid of stat cards. Identity on
the left (name → phone → email fallback chain, with the support code in monospace next to it
because the operator is reading it back). Entitlement as one state chip using the exact
`premiumState` string. Then a short row of *labelled* facts — trial days left, today's quota,
streak, last seen, app version — each of which is allowed to say "unavailable" in words.
Resist a dashboard here: this is a page header, and its whole job is to be readable in one
saccade. Activity counts belong lower, as a compact source list where each count is a link
into `timeline?sources=<key>` — they are navigation, not a metric.

**The action that follows.** Copy the support code into the ticket; jump to `incidents` if
anything failed; write a note. A real production snapshot on 2026-07-23 (fresh trial account,
minutes old) returned `premiumState: "Trial"`, `trialDaysLeft: 14`, `activity.total: 22` with
**every** count except `api_usage` at zero, `streak.currentStreak: 0` with
`available: true`, and `clients.appVersions: [{ appVersion: null, platform: null }]`. That is
the *normal* shape for most users today, and it is the shape your empty states have to look
good in: a brand-new user on a pre-1.7 build who has done nothing yet. Note the trap in that
payload — `streak.available: true` with a `0` streak is a **real** zero, whereas
`available: false` would mean "Neo4j didn't answer". Those must not render identically.

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

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/users/0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30/timeline?limit=2&sources=api_usage,orders`
— verbatim, both items shown (`limit=2`; a default call returns 50 of the same shape):

```json
{
  "items": [
    {
      "id": "api_usage:88888888-8888-4888-8888-888888888888",
      "rowId": "88888888-8888-4888-8888-888888888888",
      "source": "api_usage",
      "type": "GET /api/v1/content-doc",
      "at": "2026-09-21T14:32:49.947Z",
      "title": "GET /api/v1/content-doc → 200 (6ms)",
      "severity": "info",
      "exam": "upsc-cse",
      "data": {
        "method": "GET", "route": "/api/v1/content-doc", "status": 200,
        "durationMs": 6, "feature": null,
        "appVersion": null, "platform": null, "deviceId": null
      }
    },
    {
      "id": "api_usage:99999999-9999-4999-8999-999999999999",
      "rowId": "99999999-9999-4999-8999-999999999999",
      "source": "api_usage",
      "type": "GET /api/v1/notifications/unread-count",
      "at": "2026-09-21T14:32:49.947Z",
      "title": "GET /api/v1/notifications/unread-count → 200 (6ms)",
      "severity": "info",
      "exam": "upsc-cse",
      "data": {
        "method": "GET", "route": "/api/v1/notifications/unread-count", "status": 200,
        "durationMs": 6, "feature": null,
        "appVersion": null, "platform": null, "deviceId": null
      }
    }
  ],
  "nextCursor": "MjAyNi0wOS0yMVQxNDozMjo0OS45NDdafDk5OTk5OTk5LTk5OTktNDk5OS04OTk5LTk5OTk5OTk5OTk5OQ",
  "hasMore": true,
  "meta": {
    "sources": ["api_usage", "orders"],
    "limit": 2,
    "limitClamped": false,
    "rowsFetched": 4,
    "rowCap": 2000,
    "from": "2026-08-22T14:44:33.737Z",
    "to": "2026-09-21T14:44:33.737Z",
    "windowDays": 30,
    "retention": {
      "apiUsageRawRetentionDays": 30,
      "apiUsageCompleteFrom": "2026-08-22T14:44:33.737Z",
      "windowExceedsApiUsageRetention": false,
      "note": null
    }
  }
}
```

> 🔴 **Read the `exam` field on those two items: `"upsc-cse"`, for a user whose
> `activeExamId` is `appsc-group-1`.** That is not a bug and it is not the user's exam —
> `api_usage.exam_id` records **`req.exam` as resolved at request time**, and a client
> that sends no `X-Exam` header resolves to the default, `upsc-cse` (the "absence =
> UPSC" rule; these two rows came from a caller with no version headers at all, hence
> `appVersion`/`platform`/`deviceId` all `null`).
>
> **So never present a timeline `exam` as "the exam the user was studying".** It is "the
> exam dimension the request was served under". `orders` is the only source whose `exam`
> is a durable business fact.
>
> Note also **two items with the identical `at`** — the cursor's tiebreaker is `rowId`,
> which is why `nextCursor` encodes both. Do not de-duplicate on timestamp.

**Item fields**

| Field | Meaning |
|---|---|
| `id` | `"<source>:<rowId>"`. Stable and unique across sources — use it as the React key. |
| `rowId` | The underlying table's primary key. Also the cursor tiebreaker. |
| `source` | One of the 17 canonical values below. Drives the icon/colour. |
| `type` | Sub-kind *within* the source — `login_failed`, `POST /api/v1/quota/begin [chat_mentor]`, `PAID`, `correct`, `ISSUE/OPEN`, `razorpay.webhook`, … Not a closed enum; it is per-source and safe to display verbatim. |
| `at` | Event timestamp (UTC ISO). Whichever column that source is ordered by — see the table. |
| `title` | A pre-rendered, human-readable one-liner. **Use it.** It is built server-side per source, so the portal doesn't reimplement 17 formatters. |
| `severity` | `info` \| `warn` \| `error`. Drives colour only. `api_usage`: ≥500 → `error`, 400–499 → `warn`, else `info`. `auth_events`: `error` for the failure types listed in §3. Orders `FAILED` → `error`, `CANCELLED` → `warn`. Payments `FAILED` → `error`, `REFUNDED` → `warn`. Refunds always `warn`. Feedback `ISSUE` → `warn`, chat feedback `DOWN` → `warn`. |
| `exam` | **Present on every item, `null` on most of them.** Only three sources record an exam: `orders` (`orders.exam_id` — what was bought), `api_usage` (`api_usage.exam_id` — the `req.exam` resolved at request time) and `feedback_reports` (`feedback_reports.exam_id` — where it was filed from). Every other source is genuinely exam-less. ⚠️ It is **also `null` on history predating those columns**, which is *not* backfilled: guessing an exam for a row captured before the column existed would be indistinguishable from data. So render `null` as "—", never as "UPSC", and never filter a timeline down to one exam client-side — you would silently drop the whole pre-2026-09-21 tail. |
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

### How to use this data

**Questions it answers.** "What did this user actually do, in order?" · "Did the thing they're
describing happen at all?" · "Did the payment webhook land before or after they said they
paid?" · "Was the order created and then never granted?" The timeline is the only surface in
the product where a payment event, an app request and a support note sit on one clock.

**The decision it drives.** Whether to believe the user's account of events. Support arguments
are almost always about *ordering* — "I paid and it didn't unlock" is resolved by looking at
`orders → payments → payment_events → api_usage` in sequence, not by looking at any one of
them.

**What a good visualisation is.** A dense single-column list, reverse-chronological, grouped
under **IST day headers** (`istDate` semantics — never the UTC date of `at`). Each row is the
server-rendered `title` plus a source marker and a relative time; `data` expands in place.
Severity drives colour only, and only three colours. The source filter belongs above the list
as a multi-select that mirrors the snapshot's count keys, so clicking "22 api_usage" in the
header lands here pre-filtered. Do **not** build a swimlane or a horizontal timeline — the
operator is scanning for one event, not comparing series.

The single biggest UX risk is **`api_usage` drowning everything else.** A live production page
of 5 rows on a two-minute-old account was 5 × `api_usage`, all within the same second, because
the app fires a burst of GETs on launch. Default the filter to the *narrative* sources
(orders, payments, payment_events, refunds, auth_events, feedback, notes) and make
`api_usage` an explicit opt-in labelled as request noise. Otherwise page one is always
`GET /api/v1/quota/me`.

**The action that follows.** Copy the exact timestamp into the ticket, or switch to
`incidents` if the story is about a failure. Two mechanical rules: always render `meta.from` /
`meta.to` rather than what you asked for (the server clamps to 90 days), and always render
`meta.retention.note` when it is non-null — a quiet stretch older than 30 days is *purged*,
not idle, and an operator who reads it as idle will tell the user something false.

---

## 3. `GET /sme/users/:id/incidents`

**Purpose:** the "what went wrong" shortcut. Purely **derived** from data already captured —
no new tracking. Failures are pulled from two sources, merged newest-first, and clustered by
time proximity so an agent reads *"at 14:22 IST, 3 failures on POST /api/v1/payments/verify"* instead
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
| `exam` | slug | — | — | **Narrows the `api_usage` half only.** See the box below. **400 on an unknown slug**, not an empty feed. |

> ### `?exam=` filters half the feed, on purpose
>
> Only `api_usage` carries an exam. **`auth_events` failures are never filtered** — a sign-in
> failure happens *before* any exam is resolved, so there is nothing to filter on, and hiding
> them would remove exactly the login failures that explain the session an agent is looking at.
> Expect a filtered feed to still contain auth failures; that is correct, not a leak.
>
> ⚠️ **While `exam` is set, `api_usage` rows with a NULL `exam_id` are EXCLUDED** — that is
> every request captured before the column existed. So a filtered incident feed is *shorter
> than the truth* for any window reaching back before 2026-09-21. Label the filter with that,
> or an agent will read "no incidents in APPSC" off a window that simply predates the column.
>
> Every `sample` inside an incident carries the timeline's `exam` field (`null` where the
> source records none).

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
      "title": "at 14:22 IST, 3 failures on POST /api/v1/payments/verify (+1 other)",
      "sources": ["api_usage", "auth_events"],
      "byLabel": [
        { "label": "POST /api/v1/payments/verify", "count": 2 },
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

### How to use this data

**Questions it answers.** "Is anything actually broken for this person, or are they confused?"
— and if so, "when, how many times, and on what?" This is the endpoint that turns *"the app
isn't working"* into *"at 14:22 IST you got three 500s on the payment verify call."*

**The decision it drives.** Escalate to engineering, or don't. A cluster of 5xx on one route
is a bug report with a timestamp attached. A cluster of 402s is the paywall doing its job and
the answer is a sales conversation. A cluster of `login_failed` is an auth/OTP problem and
belongs with the OTP triage runbook, not with a feature team.

**What a good visualisation is.** A short list of *stories*, not errors — one card per
incident, headlined by the server's `title`, with `byLabel` as the breakdown and `byStatus` as
a secondary line. Five sample rows expand inside the card in the exact same component the
timeline uses (`samples[]` is deliberately `SmeTimelineItem`-shaped so you write one row
renderer). Cap the card at what fits without scrolling; `failureCount` already tells the
operator how much is hidden.

**The action that follows.** Paste `startedAtIST` + the top `byLabel` entry into the
escalation. Two things the UI must not hide: `meta.rowCapHit` / `truncatedSources` (say
"showing the most recent 250 request failures" out loud), and the fact that clustering happens
**within a page**, so a burst straddling a page boundary appears twice-halved. Raise `limit`
rather than stitching clusters client-side.

**The empty state is the common case and it is not a bug.** The verified production run on
2026-07-23 returned `incidents: []` with `rowsScanned: 0` — a healthy user genuinely has no
failures. Design that state deliberately: "No failures in the last 30 days" reads very
differently from a blank panel, and the operator is about to say it out loud to a customer.

---

## 4. `POST /sme/users/:id/notes`

**Purpose:** a staff note on a user. Stored as an `sme_audit_log` row with
`action = 'SUPPORT_NOTE'` — no new table — and it surfaces on the timeline under source
`sme_audit_log` (also reachable via `?sources=notes`).

**Request**

```json
{ "note": "Called back — Razorpay double-charge, refund raised #RF2291", "category": "refund", "author": "sme-agent@example.com" }
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
  "author": "sme-agent@example.com",
  "createdAt": "2026-07-23T09:20:44.000Z"
}
```

Unlike every other SME audit write (which is deliberately best-effort and swallows failures),
this one is written directly and **a DB failure surfaces as a 5xx** — because here the note *is*
the entire request, and silently losing it would be worse than an error. Retry on failure; the
optimistic row in the UI must be rolled back, not left on screen.

**Errors:** 400 (validation), 404 (unknown user).

### How to use this data

**The question it answers.** "Has anyone dealt with this person before, and what did they
promise them?" There is no CRM behind this product; the note trail *is* the institutional
memory for a user, and it is the only writable field on the whole surface.

**The decision it drives.** Whether to repeat a diagnosis that was already made, and whether a
refund/extension was already committed to. It also stops two operators independently
promising two different things.

**What a good visualisation is.** A compose box pinned near the top of the page (not buried at
the bottom — the operator writes the note *while* on the call), with existing notes listed
newest-first directly beneath it. Notes also appear inline on the timeline under
`sme_audit_log`; that is a feature, not a duplicate — the panel is for writing and skimming,
the timeline is for placing a note in the sequence of events.

**The action that follows.** Post, then re-fetch. Two rules: this write is **not** best-effort
like the other audit writes — a DB failure returns 5xx, so roll the optimistic row back rather
than leaving it on screen; and `author` is **self-reported and unverified** (the `x-api-key`
carries no identity), so render it as a typed-in label, never as an attested identity. If the
portal has its own logged-in user, pass it — but do not put a verified-user avatar next to it.

---

## 5. `GET /sme/users/by-support-code/:code`

**Purpose:** turn the short code a user reads out on a phone call into an account, without
asking them to spell an email address.

**Response 200**

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/users/by-support-code/1MQR-PGCT` — verbatim (that code is the `supportCode`
the snapshot in §1 returned for the same account, round-tripped):

```json
{
  "id": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
  "supportCode": "1MQR-PGCT",
  "email": "aspirant@example.com",
  "name": "A. Sharma",
  "phoneNumber": "+91XXXXXXXXXX",
  "status": "SUBSCRIBED",
  "isPremium": true,
  "premiumState": "Premium",
  "activeExamId": "appsc-group-1",
  "createdAt": "2026-09-09T11:49:43.440Z"
}
```

Feed `id` straight into §1–§4.

**`activeExamId`** (added 2026-09-21) is the exam this person uses, **nullable** — `null` means
they have never picked one. It is the first thing to read after the name: it tells the agent
which product the caller is actually talking about, before any other call.

**`premiumState` now uses that exam's trial length.** This lookup used to leave it at the 14-day
default, which was only *accidentally* right — 14 is `upsc-cse`'s length. The instant an exam
ships a different one, the first thing support saw about a caller ("Trial Ended") contradicted
the snapshot one click later. It now resolves the same per-exam length the users list and the
snapshot do, so all three agree.

⚠️ **This response does NOT carry `entitlements[]`.** It is an identification call, not an
entitlement answer. To answer "what has this person bought?", follow `id` into
`GET /sme/users/:id/snapshot` (§1) or `GET /sme/users/:id`.

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

### How to use this data

**The question it answers.** "Which account is the person on the phone?" — without asking them
to spell `sme-agent@example.com` letter by letter over a bad line.

**The decision it drives.** Nothing analytical. This is pure navigation, and it is the
**primary** entry point to the whole surface. Design it that way: a persistent input in the
global header, not a tab inside `/app-users`. Type code → land on `/app-users/:id`.

**What a good visualisation is.** A single text field with a monospace, uppercase-rendering
input and *no input mask*. The server's decoder is deliberately lenient — verified live on
2026-07-23, `54p3 6y4m` (lowercase, space instead of hyphen) resolved to the same account as
`54P3-6Y4M`. A strict mask would reject codes the server would have happily accepted, which is
the worst possible failure while a customer is reading digits aloud. Show the resolved
identity — name, email, `premiumState`, and now the **exam chip** (`activeExamId`) — for
confirmation *before* navigating, so the operator can say "is that Gatij, on APPSC?" rather than
silently opening the wrong page.

> ### 🔑 Answer sheet: "which exams has this user bought?"
>
> **Answer it from `entitlements[]` (§1) and nothing else.** Read each row's `examId` where
> `status === "LIVE"`.
>
> - ❌ **Not from `activeExamId`.** That is the exam they *study*, not the one they *bought*.
>   A user can study UPSC and hold APPSC, and vice versa; the two fields answer different
>   questions and only one of them is about money.
> - ❌ **Not from `isPremium` / `premiumState` / `subscriptionSource`.** Those are the
>   **person-level mirror**: they are true for *any* exam, and `subscriptionSource` names the
>   gateway of the **longest-lived** row — so for someone with Apple in one exam and Razorpay
>   in another, it names the wrong till.
>   ✅ **`isPremium` now honours the exam's own trial length** (fixed in this release) and
>   therefore agrees with `premiumState` and `entitlements[]`. *Before 2026-09-21* it
>   resolved an `ACTIVE` user's trial window with a fixed 14 days whatever the exam's
>   `trialDays` was, so on APPSC Group 1 (0-day trial) a lapsed trialist returned
>   `isPremium: true` beside `premiumState: "Trial Ended"` and `entitlements: []` — the
>   pre-fix capture is kept at [`SME_PORTAL_API.md` §1.1](./SME_PORTAL_API.md). It is
>   still the wrong field for a **per-exam** answer, because it is person-level by
>   design.
> - ❌ **Not from "they're SUBSCRIBED so they have everything."** That was true before exams
>   were sold separately and is now the single most expensive wrong assumption on this surface:
>   it is what made a bare revoke destroy a second exam's subscription.
>
> And the inverse: **`entitlements: []` does not mean "no access"** — a trial-only user holds no
> row at all. Their access is `premiumState: "Trial"` plus `trialEndsAt`.

**The action that follows.** Navigate. Handle three failures distinctly: `400` = "that isn't a
valid code, ask them to read it again" (the message names the offending symbol), `404` = "no
account has that code", `409` = "ambiguous — ask for their email or phone instead". Do **not**
build a candidate picker for the 409; `candidates` is stripped by the exception filter and
there is nothing to populate it with.

---

## Appendix A — the client-event sink (`POST /diagnostics/events`)

Not an SME endpoint, but it is where half the timeline's `auth_events` rows come from, so the
portal team should know how they arrive.

- **Routes:** `POST /api/v1/diagnostics/events` **and** `POST /api/v1/diagnostics/auth-events` —
  two paths, **one handler, identical semantics**. `events` is the honest name now that the sink
  carries general client events; `auth-events` is kept **forever** because 1.6/1.7 builds already
  in the field call it and will never be updated. (Current store build is **2.0** — Android 27,
  iOS 2.0 build 3, released 2026-09-17 — and it calls `events`.)
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
| Auth lifecycle | `login_attempt`, `login_success`, `login_failed`, `otp_send_failed`, `otp_verify_failed`, `refresh_failed`, `forced_logout` | **Yes.** `otp_send_failed` also has a server-side emitter, so it is the one type with full coverage; the rest are client-emitted and so follow the 2.0 adoption curve below. |
| Client errors | `client_error` | **Yes** — emitted by app 2.0 (released 2026-09-17); older builds emit nothing |
| Purchase funnel | `paywall_viewed`, `upgrade_tapped`, `checkout_opened`, `checkout_abandoned`, `purchase_failed` | **Yes** — emitted by app 2.0 (released 2026-09-17); older builds emit nothing |
| App lifecycle | `app_opened`, `app_backgrounded` | **Yes** — emitted by app 2.0 (released 2026-09-17); older builds emit nothing |
| Chat pointer | `chat_conversation_started` | **Yes** — but written **server-side** by `POST /quota/finalize`, not by the sink, and only when the app sends a `conversationId`, which app 2.0 does (older builds do not). `metadata = { feature, conversationId }`, conversationId truncated to 128 chars. |

Volume on every client-emitted type grows with **2.0 adoption**, not with time: a device still
on 1.7/1.8/1.9 contributes nothing and never will. Read a low count as partial coverage of the
install base, never as "this did not happen".

---

## Appendix B — schema changes behind this surface

Migration `prisma/migrations/20260723170000_activity_trail_capture/migration.sql`. All additive
and nullable; hand-applied (the `_prisma_migrations` table is not a reliable source of truth on
this project — verify via `information_schema`, not Prisma's migration state).

| Change | Why it matters to the portal |
|---|---|
| `api_usage.feature TEXT` | Sub-route discriminator for `/quota/begin` only. **Populated from the deploy date onward** — older rows are `null` even for chat calls. A `null` feature on a `/quota/begin` row means "before this shipped", not "unknown feature". |
| `auth_events.metadata JSON` | The per-event payload. `null` on every row written before the deploy. |
| `device_tokens.permission_granted BOOLEAN` | Populated by app 2.0 (released 2026-09-17); `null` on tokens from any older build, permanently. **Never render `null` as denied.** |
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
- `api_usage.route` keeps the `/api/v1` prefix. Strip it yourself if you group by module.
- `premiumState` is `Premium` / `Trial` / `Trial Ended` / `Churned` / `Downloaded` — title
  case, one of them with a space. Not SCREAMING_CASE.

---

Related: [SME_USER_DETAIL_PAGE_GUIDE.md](./SME_USER_DETAIL_PAGE_GUIDE.md) (the UI this
contract is for) · [SME_INSIGHTS_API.md](./SME_INSIGHTS_API.md) (the population-level view of
the same `api_usage` / `auth_events` tables) ·
[SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) ·
[SME_PORTAL_API.md](./SME_PORTAL_API.md) (the `/app-users` roster this page links from) ·
[archive/WHAT_CHANGED_2026-07-23.md](./archive/WHAT_CHANGED_2026-07-23.md) (archived)
