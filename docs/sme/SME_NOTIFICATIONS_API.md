# SME — Push Notifications API Guide

> Changed 2026-09-21 — per-exam broadcast, validated segment exam (+ fix), survey targetExams, feedback ?exam= (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

End-to-end reference for the **SME portal** to send push notifications to PrepMonkey
users — to **one user**, to **everyone**, to **one exam mode**, or to a **cohort**
(premium / trial / free).
Same server-to-server model as the rest of the SME surface: we expose the endpoints,
the SME team builds the UI.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `BASE_URL` = `https://app.stanzasoft.ai`)
**Authentication:** `x-api-key: <API_KEY_SECRET>` header on **every** request.
**Swagger:** `/api/docs` (group **SME**) documents these live with the `x-api-key` control.

- Missing/invalid key → **401** `{ "message": "Invalid or missing API key" }`.
- All routes use the `/api/v1/` global prefix.
- These are the **audited** wrappers (every send is written to `sme_audit_log`). Prefer
  them over the raw `/notifications/send*` endpoints.

---

## 0. The mental model (read this first)

A push reaches a phone only if **all** of these are true:
1. The app is installed from a build that has the push code (shipped since the June
   release — already live for existing users).
2. The user **granted notification permission** (asked once on first dashboard arrival).
3. The device **registered an FCM token** (happens after login) — and for *topic*
   sends, the device has **opened the dashboard at least once** (that is when it
   subscribes to its topics).

Delivery itself is **best-effort** (FCM): a `successCount` is returned, not a
read-receipt. Dead/expired tokens are auto-deactivated on send.

Every send is also **persisted to the in-app notification feed**, so a user who was
offline still sees it in the bell feed when they reopen the app.

---

## 1. Send to ONE user — `POST /sme/users/:id/notify`

`:id` is the user's **UserAuth id** (the `id` from `GET /sme/users`). Sends to *all of
that user's* active devices (iOS / Android / web).

**Request**
```
POST /api/v1/sme/users/0d2f.../notify
x-api-key: <API_KEY_SECRET>
Content-Type: application/json
```
```json
{
  "title": "Your evaluation is ready",
  "body": "Tap to view your Mains answer feedback.",
  "type": "mains_question",
  "id": "mq_abc123"
}
```
| Field | Req | Notes |
|---|---|---|
| title | ✅ | ≤ 120 chars |
| body | ✅ | the message |
| type | ➖ | deep-link type for tap routing (see §4). Default `home` |
| id | ➖ | entity id forwarded in the payload (used by some `type`s) |
| params | ➖ | extra deep-link arguments for the few `type`s that need more than an id — flat `{"key":"value"}` object, see §4.1 |

**Response 200**
```json
{ "successCount": 2, "failureCount": 0 }
```
- **404** `{ "message": "User not found" }` if the id doesn't exist.
- `successCount: 0` with no error = the user has **no active device tokens** (never
  logged in on a push-capable build, or revoked permission). Not an error.

**curl**
```bash
curl -X POST "$BASE_URL/api/v1/sme/users/$USER_ID/notify" \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{"title":"Hi","body":"Welcome back!","type":"home"}'
```

---

## 2. Broadcast to EVERYONE — or to ONE EXAM — `POST /sme/notifications/broadcast`

Sends to the FCM `all_users` topic — i.e. every device subscribed to it. This is a
**single FCM topic send** (cheap, instant), not a per-user fan-out.

**Request**
```
POST /api/v1/sme/notifications/broadcast
x-api-key: <API_KEY_SECRET>
```
```json
{ "title": "New mock test live!", "body": "Attempt the 2026 Prediction Test now.", "type": "home" }
```
Same body fields as §1 (no `:id` path param; `type`/`id`/`params` optional, `type` default `home`),
plus one optional field:

| Field | Req | Notes |
|---|---|---|
| exam | ➖ | exam slug, e.g. `appsc-group-1`. Sends to the `exam_<slug>` topic instead of `all_users`. Omit for the app-wide broadcast |

**Response 200** (both paths)
<!-- shape verified against code @c0a8fe5 — NOT captured on staging, see note -->
```json
{
  "messageId": "projects/prepmonkey-db925/messages/0:1700000000%...",
  "topic": "all_users"
}
```

> ⚠️ **Not captured from staging, and it cannot be.** Staging's Firebase service account
> cannot mint an access token — any real send there fails at the FCM call with a **500**
> mentioning `iam.serviceAccounts.getAccessToken`, so there is no `messageId` to capture.
> **This success shape was verified on the production credential path on 2026-09-17 for
> the all-users case** (`topic: "all_users"`); the exam branch differs only in the
> `topic` string. Everything on this route that runs *before* the FCM call — notably the
> `exam` validation in §2.1 — **was** captured on staging and is shown there.
A topic send returns an FCM **message id**, not per-user counts (FCM fans out to
subscribers asynchronously). There is no count of how many devices it reached.

`topic` is **new and present on both paths** — `all_users` when `exam` is omitted,
`exam_<slug>` when it is not. It is deliberately not conditional: the portal can state
the audience it just reached by reading the response, without branching on its own
request body.

### 2.1 Scoping a broadcast to one exam mode

```json
{ "title": "Group-1 prelims key is out", "body": "Check your score now.",
  "type": "home", "exam": "appsc-group-1" }
```
<!-- shape verified against code @c0a8fe5 — success body not capturable on staging (§2) -->
```json
{
  "messageId": "projects/prepmonkey-db925/messages/0:1700000000%...",
  "topic": "exam_appsc-group-1"
}
```

**The rejection path *was* captured.** `POST /sme/notifications/broadcast` with
`{"title":"x","body":"y","type":"home","exam":"appsc-grp-1"}` on staging, 2026-09-21
(backend `f6329e6`) — the guard runs **before** the FCM call, so nothing was sent:
<!-- captured from staging 2026-09-21, backend f6329e6 -->
```json
{
  "success": false,
  "message": "Unknown exam \"appsc-grp-1\". Create it via POST /sme/exams first.",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:38:25.811Z",
  "path": "/api/v1/sme/notifications/broadcast",
  "method": "POST"
}
```

And the topic name itself is captured, from the app-facing resolver in §5: a live
`GET /notifications/topics` for a user whose `active_exam_id` is `appsc-group-1` returns
**`"exam_appsc-group-1"`** — hyphen intact — which is exactly the string a broadcast must
address.

- The topic name is **`exam_` + the slug, verbatim** — hyphens are kept
  (`exam_appsc-group-1`, `exam_upsc-cse`), not collapsed to underscores like the
  `aspirant_*` / `medium_*` topics. The slug is the same literal used by `?exam=`,
  `X-Exam` and `user_profiles.active_exam_id`, so there is no second spelling to learn.
- **An unknown slug is a 400**, not a quiet send into the void:
  `Unknown exam "<slug>". Create it via POST /sme/exams first.` (A topic send to a
  nonexistent topic returns a perfectly good `messageId` and reaches nobody, forever —
  which is why this one is validated rather than coerced.)
- ⚠️ **`exam: "upsc-cse"` also reaches everyone who has never used the exam picker or
  the home switcher.** An unset preference means UPSC everywhere in this API, and the
  topic resolver follows the same rule (§5).
- **Feed persistence is unchanged in shape:** a broadcast writes **one**
  `topic_notifications` row keyed by the topic it addressed, and the bell feed merges
  in the rows whose topic is in the reader's own resolved topic list. So an exam
  broadcast is in-app visible to **exactly** the users who are on `exam_<slug>` — the
  feed needs no exam column of its own, the topic *is* the scope.

**Reach caveat:** only devices that have **opened the dashboard** (with permission
granted) are subscribed to `all_users`. Brand-new / never-opened installs are not yet
on the topic.

⚠️ **Coverage of the new `exam_*` topics grows over time.** A device joins its exam
topic the next time it loads the dashboard and re-syncs `GET /notifications/topics`
(§5) — **no app release is involved**, but a device that has not opened the app since
this shipped is not on its exam topic yet. In the first days after rollout an exam
broadcast reaches a *subset* of that exam's users; `all_users` is unaffected. There is
no count to check this against — compare it against how many users the segment send
(§3) reports for the same exam.

---

## 3. Send to a COHORT — `POST /sme/notifications/segment`

Targets **premium / trial / free**. Resolved by a **live Postgres query at send time**
(NOT FCM topics), then fanned out per-user in batches of 50.

**Request**
```
POST /api/v1/sme/notifications/segment
x-api-key: <API_KEY_SECRET>
```
```json
{ "segment": "premium", "title": "Premium tip", "body": "Unlimited PYQs await.", "type": "home" }
```
| Field | Req | Notes |
|---|---|---|
| segment | ✅ | `premium` \| `trial` \| `free` |
| title / body | ✅ | as above |
| type / id / params | ➖ | as above |
| exam | ➖ | exam slug — narrows the cohort to one exam mode. Omit to reach the whole cohort |

**Segment definitions (authoritative — live state, not the raw status column):**
| segment | who |
|---|---|
| `premium` | `status = SUBSCRIBED` **and** not expired (`premium_expires_at` null or in the future) |
| `trial` | `status = ACTIVE` **and** `trial_ends_at` still in the future — i.e. the trial length configured for the exam they started on, **not a fixed 14 days**. Rows with no `trial_ends_at` (pre-dating the column) fall back to "created within 14 days and never paid", so legacy accounts resolve exactly as they always did |
| `free` | everyone else |

The three cohorts are **person-level**: they read the `user_auth` mirror, so "premium"
means *holds premium somewhere*, not *holds premium in this exam*. `exam` is the other
axis — see below.

**Response 200**
<!-- shape verified against code @c0a8fe5 — NOT captured on staging, see note -->
```json
{
  "segment": "premium",
  "exam": null,
  "matchedUsers": 312,
  "successCount": 298,
  "failureCount": 14
}
```

> ⚠️ **Not captured, same reason as §2:** a real fan-out on staging dies at the FCM call
> (500, `iam.serviceAccounts.getAccessToken`). Verified on the production credential path
> on 2026-09-17 for the all-users case. The `exam` validation below **was** captured.
`matchedUsers` = users in the cohort; `successCount`/`failureCount` are device-level
(a matched user with no active device adds 0 to both).
`exam` is **new** — the normalised slug that was actually applied, or `null` on the
un-scoped call (which is every call that existed before this parameter).

⚠️ **Synchronous:** the HTTP call blocks until the whole cohort is sent (batched 50 at a
time). For very large cohorts this can take a while — set a generous client timeout.

### 3.1 Narrowing to one exam — `exam`

```json
{ "segment": "trial", "title": "Your Group-1 trial ends tomorrow",
  "body": "Keep your access.", "type": "premium", "exam": "appsc-group-1" }
```
<!-- shape verified against code @c0a8fe5 — success body not capturable on staging (§3) -->
```json
{ "segment": "trial", "exam": "appsc-group-1", "matchedUsers": 48,
  "successCount": 44, "failureCount": 4 }
```

**The unknown-slug rejection was captured** — it runs before the cohort is even resolved,
so nothing is queried and nothing is sent. `POST /sme/notifications/segment` with
`{"segment":"premium","title":"x","body":"y","type":"home","exam":"appsc-grp-1"}` on
staging, 2026-09-21 (backend `f6329e6`):
<!-- captured from staging 2026-09-21, backend f6329e6 -->
```json
{
  "success": false,
  "message": "Unknown exam \"appsc-grp-1\". Create it via POST /sme/exams first.",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:38:26.106Z",
  "path": "/api/v1/sme/notifications/segment",
  "method": "POST"
}
```

- Matching is on **`user_profiles.active_exam_id`** — who the person *is* (their last
  switcher / picker choice), not what they hold.
- ⚠️ **`exam: "upsc-cse"` also matches every user who has never touched the picker or
  the switcher** (`active_exam_id` NULL, or no profile row at all). An unset preference
  means UPSC, and without that rule a UPSC-scoped send would reach almost nobody. Same
  rule as `GET /sme/users?exam=`.
- **Unknown slugs are now a 400** — `Unknown exam "<slug>". Create it via POST
  /sme/exams first.` Previously a typo resolved to an empty cohort and reported a
  successful send to nobody, which is indistinguishable from a real cohort that happens
  to be empty.

> 🔴 **FIXED 2026-09-21 — a `trial` or `premium` send scoped to `upsc-cse` used to hit
> the wrong people.**
>
> The exam clause was merged into the cohort query by spreading it over the top-level
> `where`. For the **default** exam that clause carries its own `OR` (the NULL rule
> above), and the `premium` and `trial` cohorts each have an `OR` of their own — so the
> exam clause **replaced** the cohort's `OR`, silently:
>
> - `segment:"trial"` + `exam:"upsc-cse"` lost the in-window test entirely, leaving
>   `status = ACTIVE` — **which is every free account.** A "your trial is ending" push
>   scoped to UPSC went to the entire free base.
> - `segment:"premium"` + `exam:"upsc-cse"` lost the "not expired" half and also
>   reached lapsed subscribers.
>
> Both are correct from this release (the clause is composed with `AND`). `free` was
> never affected, and no *un-scoped* send (`exam` omitted) was ever affected.
> **If you sent a scoped premium/trial push before this release, its `matchedUsers` was
> wrong — do not use it as a cohort size.** Non-default exams (`appsc-group-1`, …) were
> also unaffected: their clause has no `OR`.

Everything else about the three segments is unchanged.

---

## 4. Deep-link `type` contract (tap routing)

`type` (+ optional `id`, + optional `params`) is forwarded in the FCM **data payload**;
when the user taps, the app routes via `DeepLinkMapper.fromData`. **Only these values
route** — anything else lands on **Home**:

### 4.1 Extra arguments — `params`

A few screens need more than one id (a practice list needs *which filter*, a document
list needs *which subject*). Those take a **`params`** object alongside `type`:

```json
{
  "title": "Revise 2025 Prelims",
  "body": "24 questions from last year's paper.",
  "type": "practice_list",
  "params": { "filterType": "year", "filterValue": "2025" }
}
```

- Flat **string → string** only (no nesting, no numbers, no booleans — quote them).
- **≤ 10 keys**, key ≤ 40 chars, value ≤ 256 chars, **≤ 1 KB** serialized.
  (FCM's tightest data limit is 2 KB on a *topic* send, and `title`/`body`/`type`/`id`
  already spend part of it. Over the cap → **400**, with the offending field named.)
- **Optional everywhere.** Omit it and every `type` behaves exactly as it always has.
- Unknown keys are ignored by the app; a missing *required* param degrades to the
  fallback in the table below (never an error screen).
- The universal-link equivalent is the **query string**:
  `/open/practice_list?filterType=year&filterValue=2025`.

### 4.2 Routable keys

`id` column: ✅ = required (missing → the **fallback** column), ➖ = not used,
"optional" = changes the destination when present.

| `type` | `id` | `params` | opens | if `id`/`params` missing | universal link |
|---|---|---|---|---|---|
| `home` (default) | ➖ | — | Home / dashboard | — | `/open/home` |
| `daily_task` | ➖ | — | Daily tasks (Home) | — | `/open/daily_task` |
| `chat` | ➖ | — | Chat | — | `/open/chat` |
| `chat_expert` | ➖ | — | Chat, **Expert (SME) mode** preselected | — | `/open/chat/expert` |
| `mains` | ➖ | — | PYQ tab → **Mains** landing | — | `/open/mains` |
| `mains_question` | ✅ questionId | — | that Mains question detail | Mains landing | `/open/mains/<id>` |
| `reel` | optional reelId | — | that reel, playing | Reels feed | `/open/reel[/<id>]` |
| `reelblog` | ✅ reelId | — | that reel's blog article | Reels feed | `/open/reelblog/<id>` |
| `pyq` | ➖ | — | PYQ / Practice tab | — | `/open/pyq` |
| `pyq_question` | ✅ questionId | — | that PYQ question (read-only) | PYQ tab | `/open/pyq/<id>` |
| `simulation` | ✅ simulationId | — | Library → Simulation, highlighted (no auto-start) | Library | `/open/simulation/<id>` |
| `doc` | ✅ documentId | — | that study document | Library | `/open/doc/<id>` |
| `library` | ➖ | — | Library / My Content | — | `/open/library` |
| `flashcards` | ➖ | — | Chat with the **flashcard generator** open | — | `/open/flashcards` |
| `mnemonics` | ➖ | — | Chat with the **mnemonic generator** open | — | `/open/mnemonics` |
| `saved` | ➖ | — | Saved questions | — | `/open/saved` |
| `premium` | ➖ | — | Upgrade / paywall (campaign-aware) | — | `/open/premium` |
| `offer` | ✅ campaign code | — | that campaign's paywall | standard paywall | `/open/offer/<code>` |
| `report` | ✅ reportId | — | that feedback report's detail | Home | `/open/report/<id>` |
| `survey` | ✅ surveyId | — | the survey runner | Home | `/open/survey/<id>` |
| `profile` | ➖ | — | Profile | — | `/open/profile` |
| `my_account` | ➖ | — | My Account | — | `/open/my_account` |
| `my_activity` | ➖ | — | My Activity (streak + reading) | — | `/open/my_activity` |
| `notification_feed` | ➖ | — | the in-app notification feed (bell) | — | `/open/notification_feed` |
| `faq` | ➖ | — | FAQ | — | `/open/faq` |
| `terms` | ➖ | — | Terms & Conditions | — | `/open/terms` |
| `phone_verify` | ➖ | — | phone-number verification (enter number → OTP) | — | `/open/phone_verify` |
| `help_feedback` | ➖ | — | Help & Feedback | — | `/open/help_feedback` |
| `feedback_form` | optional | `formType` | the report form, `ISSUE` or `FEATURE` | `ISSUE` | `/open/feedback_form/FEATURE` |
| `my_reports` | ➖ | — | My Reports (list) | — | `/open/my_reports` |
| `survey_list` | ➖ | — | Surveys (list) | — | `/open/survey_list` |
| `weak_topics` | ➖ | — | "Practice your mistakes" topic list | — | `/open/weak_topics` |
| `replay_session` | optional subject | `subject`, `topic` | mistake replay; no params = **all** outstanding mistakes | replays everything | `/open/replay_session/History` |
| `practice_list` | ➖ | `filterType` ✅, `filterValue` ✅, `examType` | a filtered PYQ question list | **PYQ tab** | `/open/practice_list?filterType=year&filterValue=2025` |
| `add_task` | optional taskId | `taskId` | the task editor; no id = blank "add task" form | blank form | `/open/add_task` |
| `doc_list` | optional subject | `subject` | that subject's document list | Library | `/open/doc_list/History` |
| `saved_flashcards` | ➖ | — | saved **flashcards list** (≠ `flashcards`, which creates one) | — | `/open/saved_flashcards` |
| `saved_mnemonics` | ➖ | — | saved **mnemonics list** | — | `/open/saved_mnemonics` |
| `saved_reels` | ➖ | — | saved reels / Updates list | — | `/open/saved_reels` |
| `simulation_review` | ✅ **attemptId** | `name` | question-by-question review of that attempt | Library | `/open/simulation_review/<attemptId>` |

**Param values that are validated, not passed through:**
| param | used by | accepted | anything else |
|---|---|---|---|
| `filterType` | `practice_list` | `year` \| `subject` | → PYQ tab |
| `filterValue` | `practice_list` | e.g. `2025`, `History` | → PYQ tab |
| `examType` | `practice_list` | `prelims` \| `mains` | → `prelims` |
| `formType` | `feedback_form` | `ISSUE` \| `FEATURE` | → `ISSUE` |
| `subject` / `topic` | `replay_session`, `doc_list` | free text | omitted = broader scope |
| `name` | `simulation_review` | display name | the app fetches the real name |
| `taskId` | `add_task` | a task id | omitted = new task |

Anything not in this table falls back to **Home**. `report` and `survey` are sent
automatically by the feedback module — `report` on a status change or SME reply
(`id` = reportId), `survey` by `POST /sme/surveys/:id/notify` (`id` = surveyId). A
survey deep-link to an expired/replaced survey lands on a graceful "no longer
available" state, never an error screen. `type` is a free string on the send
API (no backend change to use a new one).

⚠️ **Version floor.** A `type` routes only on **app builds ≥ the release that shipped
it** — older installs fall back to Home for that *push* type (universal links degrade
more gracefully). Everything from `profile` downwards in the table, plus `params`
itself, ships in the **August 2026** release; older installs ignore `params` entirely
and route on `type`/`id` alone. Adding a brand-new target that isn't an existing app
screen still needs an app release.

✅ **The in-app feed carries `params` too** (corrected 2026-08-26 — this section previously
said it did not). `notification_history.params` / `topic_notifications.params` have existed
since migration `20260821020000_notification_params`, the feed response returns them, and
`NotificationFeedScreen` re-encodes them through the same `DeepLinkMapper.fromData` the push
tap uses. A `params`-dependent target such as `practice_list` routes identically from the bell
feed and from the system shade.

⚠️ Still true: an install older than the August 2026 release ignores `params` entirely and
routes on `type`/`id` alone, from either surface.

### 4.2b Per-user opt-outs now exist — and they apply to YOUR sends too

Users have four category toggles (Payment · Streak · Progress · Content) plus a "pause
everything for N days" switch, exposed to the app as:

```
GET   /notifications/preferences        (JWT)
PATCH /notifications/preferences        (JWT)
POST  /notifications/tapped             (JWT) — reports a real notification open
```

⚠️ **A user who has never opened that screen has every category ON.** Absence of a stored
preference is not an opt-out — on the day this shipped, that was the entire installed base.

The three ad-hoc SME send routes in this document (`/sme/users/:id/notify`,
`/sme/notifications/broadcast`, `/sme/notifications/segment`) are **direct sends and do not
consult preferences, quiet hours or the daily cap.** They are the manual override and behave
exactly as they did before. Anything that should respect a user's choice belongs in a
notification rule instead — see
[SME_NOTIFICATION_RULES_API.md](./SME_NOTIFICATION_RULES_API.md).

### 4.3 Deliberately NOT routable

These screens exist but cannot be reached by a link, because they read state that only
the in-app journey produces — a deep link would land on a blank or "unavailable" screen:

| screen | why not |
|---|---|
| Mains **write answer** | reads the question the detail screen loaded; would open a blank editor |
| Mains **evaluation result** | renders the evaluation from the just-finished session; cold = "No evaluation available" |
| **Mock test** (+ its result / review) | `count` has no default, and the result screen reads the in-memory attempt |
| **Simulation test** | starting it creates/resumes a timed attempt — a deliberate product decision that `simulation` highlights the card instead |
| **Simulation loading** / **result** | transient; the result screen reads the just-submitted attempt (`simulation_review` is the durable equivalent) |
| **Simulation review detail** | a pager inside the review list; enter via `simulation_review` |
| **Practice question** pager | reads the list `practice_list` loads; use `practice_list` |
| **Saved-reels player** | reads the list `saved_reels` loads; use `saved_reels` |
| **Time allocation edit** | pre-filled with the user's *current* times, which the server does not send; wrong defaults could be saved over the real ones |
| **Notification settings** | a static "No notifications yet" placeholder — use `notification_feed` |
| **OTP verification** | needs a live OTP send; use `phone_verify`, which starts that flow |
| **Delete account** | never an appropriate destination for a push |

---

## 5. How "premium vs free" is decided — and where (the FAQ)

Two **separate** mechanisms; do not conflate them:

**A) Topic subscription (used by `broadcast`)** — `topic.service.ts → resolveTopicsForUser()`
- Server computes each user's desired topics from their profile: `all_users` +
  `tier_premium`/`tier_free` + **`exam_<slug>`** + `aspirant_*` + `target_<year>` +
  `medium_<lang>`.
- Tier rule here: **paid and current, or inside the free-trial window → `tier_premium`**,
  else `tier_free` (**trial users land in `tier_premium`**). Corrected 2026-09-21: this
  used to read `SUBSCRIBED || ACTIVE`, and `ACTIVE` is the default status on *every* free
  account — so a lapsed trial, and anyone who had not opened the app since their trial
  ended, were both being pushed as `tier_premium`.
- **Exactly one `exam_<slug>` per user, always present.** It is the user's
  `active_exam_id`; a NULL value — or no profile row at all — yields **`exam_upsc-cse`**,
  because absence means UPSC everywhere in this API and those users were genuinely served
  UPSC. Hyphens are preserved (`exam_appsc-group-1`); this topic is deliberately not
  passed through the `aspirant_*`/`medium_*` normaliser, which would rewrite it to
  `exam_appsc_group_1`.
- The topic set is **person-level, not per-exam**, apart from that one key: a device
  subscribes to topics, not to an exam, so `tier_premium` still means "holds premium
  somewhere".
- The **app applies it**: on each dashboard load it fetches `GET /notifications/topics`
  and subscribes/unsubscribes via FCM to match. No app release is needed to change
  segments — edit `resolveTopicsForUser` and clients pick it up on next sync. That is
  how `exam_<slug>` reaches the installed base, and also why its coverage grows as users
  open the app rather than arriving complete on deploy day (§2.1).

**Example — `GET /notifications/topics` (JWT, app-facing):**
<!-- captured from staging 2026-09-21, backend f6329e6 -->
Verbatim, for a real staging account whose `active_exam_id` is `appsc-group-1`:

```json
{ "topics": ["all_users", "tier_premium", "exam_appsc-group-1",
             "aspirant_full_time", "medium_english"] }
```

> **Two things that capture settles:**
>
> * **`exam_appsc-group-1` is present, hyphen intact, sitting third** — this is the live
>   proof that the new topic reaches an installed client with no app release, purely by
>   re-syncing this endpoint.
> * **There is no `target_*` topic**, because this account's `user_profiles.target_year`
>   is `null`. `aspirant_*`, `target_*` and `medium_*` are each emitted **only when their
>   profile column is set**, so the list length varies per user — `all_users`,
>   `tier_*` and `exam_*` are the only three that are always present. A client that
>   assumes a fixed six-topic list will mis-diff its subscriptions.
>
> ✅ **`tier_premium` now honours the exam's own trial length** — fixed in this release,
> together with the SME `isPremium` field it shares a helper with
> ([`SME_PORTAL_API.md` §1.1](./SME_PORTAL_API.md)). *Before 2026-09-21*, an `ACTIVE`
> user whose `trial_ends_at` was NULL had their window computed as `created_at + 14 days`
> whatever the exam's `trialDays` was, so on APPSC Group 1 (trial **0** days) a lapsed
> trialist stayed on `tier_premium` — and received premium-audience pushes — for the
> first 14 days after signup. Devices pick the correction up on their next
> `GET /notifications/topics` sync; no app release is involved.
>
> The remaining `tier_premium` caveat is the **intended** one, not a bug: it still
> includes users in a *live* trial, while the `premium` **segment** (§3) does not.

**B) Segment fan-out (used by `segment`)** — `sme-user.service.ts → resolveSegment()`
- No topics. A live DB query selects the cohort (see §3 table), then per-user send.

⚠️ **They disagree on trial:** the `tier_premium` *topic* includes trial users; the
`premium` *segment* does not. **For accurate premium/trial/free targeting, use the
`segment` endpoint (§3).** There is still **no SME endpoint** that pushes the
`tier_premium`/`tier_free` topics directly — `broadcast` addresses `all_users` or, with
`exam`, `exam_<slug>`, and nothing else.

⚠️ **They also count the exam differently, and it matters.** `broadcast?exam=` is a
topic send: it reaches devices that have *synced since this shipped*. `segment?exam=` is
a live query: it reaches every matching user's registered devices, synced or not. For
anything time-critical in a non-UPSC exam, prefer `segment` until topic coverage has had
a few days to fill in. Both resolve NULL `active_exam_id` to `upsc-cse`, so the two agree
on *who* belongs to an exam.

---

## 6. Caveats checklist

- **Permission + dashboard:** topic sends (`broadcast`) only reach devices that opened
  the dashboard with permission granted. Per-user (`/notify`) and `segment` sends reach
  any device with a registered active token (also requires permission).
- **`exam_<slug>` coverage grows, it does not arrive complete.** A device joins its exam
  topic on its next dashboard sync of `GET /notifications/topics`. No app release is
  needed, but an exam broadcast in the first days after rollout reaches a subset (§2.1).
  `all_users` is unaffected.
- **`exam` is validated on both send routes** — unknown slug → **400**
  `Unknown exam "<slug>". Create it via POST /sme/exams first.` Neither route will
  quietly send to nobody any more.
- **`exam` is the "who they ARE" axis** (`active_exam_id`), not "what they HOLD"
  (`user_exam_entitlements`). `exam: "upsc-cse"` additionally matches every user with no
  stored preference; the tier cohorts stay person-level.
- **A scoped `premium`/`trial` send before 2026-09-21 targeted the wrong cohort** (§3.1)
  — treat any `matchedUsers` recorded from one as meaningless.
- **iOS foreground:** notification-type messages are not auto-shown while the app is in
  the foreground (handled by the in-app feed instead). Backgrounded/closed = shown.
- **Best-effort:** `successCount` ≠ delivered-and-seen. No read receipts.
- **Dead tokens** are auto-deactivated when FCM reports `registration-token-not-registered`.
- **Persistence:** every send is written to the in-app feed (`notification_history` for
  per-user/segment; `topic_notifications` for broadcast) so users see missed ones.
- **Amounts/rupees** etc. are irrelevant here (no money in this module).
- **`params` is additive:** omitting it reproduces the exact payload sent before it
  existed, and app builds that predate it ignore the key rather than failing.
- **Auditing:** per-user + segment sends are recorded in `sme_audit_log` (`USER_NOTIFY`).

---

## 7. Implementation checklist (SME portal)

1. Store `API_KEY_SECRET` **server-side only** — never ship it to a browser/client.
2. **One user:** find the user via `GET /sme/users?search=...` → take `id` →
   `POST /sme/users/:id/notify`.
3. **Everyone:** `POST /sme/notifications/broadcast`. **One exam:** the same call with
   `exam: "<slug>"`; read `topic` back from the response and show it as the audience.
4. **Cohort:** `POST /sme/notifications/segment` with `segment: premium|trial|free`,
   optionally `exam`. Offer the exam picker from `GET /sme/exams` — a free-text slug
   field will earn 400s.
5. Pick a `type` from §4.2 (default `home`); pass `id` when the type needs it, and
   `params` for the few that take extra arguments (§4.1). Validation is strict —
   an over-cap or non-string `params` is a **400**, not a silently dropped field.
6. Read the response counts; treat `successCount:0`/low counts as "few/no active
   devices", not an error.
7. For `segment`, use a long client timeout (synchronous fan-out).

---

## 8. Quick reference

| Goal | Endpoint | Audience source | Returns |
|---|---|---|---|
| One user | `POST /sme/users/:id/notify` | that user's devices | `{successCount, failureCount}` |
| Everyone | `POST /sme/notifications/broadcast` | FCM `all_users` topic | `{messageId, topic}` |
| One exam mode | `POST /sme/notifications/broadcast` + `exam` | FCM `exam_<slug>` topic | `{messageId, topic}` |
| Premium / trial / free | `POST /sme/notifications/segment` | live DB query | `{segment, exam, matchedUsers, successCount, failureCount}` |
| …narrowed to one exam | `POST /sme/notifications/segment` + `exam` | live DB query on `active_exam_id` | as above, `exam` echoed |

All require `x-api-key: <API_KEY_SECRET>`. Prod base: `https://app.stanzasoft.ai/api/v1`.
