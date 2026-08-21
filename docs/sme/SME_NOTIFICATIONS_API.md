# SME — Push Notifications API Guide

End-to-end reference for the **SME portal** to send push notifications to PrepMonkey
users — to **one user**, to **everyone**, or to a **cohort** (premium / trial / free).
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

## 2. Broadcast to EVERYONE — `POST /sme/notifications/broadcast`

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
Same body fields as §1 (no `:id` path param; `type`/`id`/`params` optional, `type` default `home`).

**Response 200**
```json
{ "messageId": "projects/prepmonkey-db925/messages/0:1700000000%..." }
```
A topic send returns an FCM **message id**, not per-user counts (FCM fans out to
subscribers asynchronously). There is no count of how many devices it reached.

**Reach caveat:** only devices that have **opened the dashboard** (with permission
granted) are subscribed to `all_users`. Brand-new / never-opened installs are not yet
on the topic.

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

**Segment definitions (authoritative — live state, not the raw status column):**
| segment | who |
|---|---|
| `premium` | `status = SUBSCRIBED` **and** not expired |
| `trial` | `status = ACTIVE`, never paid, within 14 days of signup |
| `free` | everyone else |

**Response 200**
```json
{ "segment": "premium", "matchedUsers": 312, "successCount": 298, "failureCount": 14 }
```
`matchedUsers` = users in the cohort; `successCount`/`failureCount` are device-level
(a matched user with no active device adds 0 to both).

⚠️ **Synchronous:** the HTTP call blocks until the whole cohort is sent (batched 50 at a
time). For very large cohorts this can take a while — set a generous client timeout.

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

⚠️ **The in-app feed carries `type` + `id` only.** A tap in the bell feed re-routes from
the stored notification, which does **not** persist `params`. A `params`-dependent
target (`practice_list`) therefore lands on its fallback (the PYQ tab) when opened from
the feed rather than from the push itself. Prefer `type`+`id` targets when the feed tap
matters as much as the push tap.

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
  `tier_premium`/`tier_free` + `aspirant_*` + `target_<year>` + `medium_<lang>`.
- Tier rule here: `SUBSCRIBED || ACTIVE → tier_premium`, else `tier_free`
  (**trial users land in `tier_premium`**).
- The **app applies it**: on each dashboard load it fetches `GET /notifications/topics`
  and subscribes/unsubscribes via FCM to match. No app release is needed to change
  segments — edit `resolveTopicsForUser` and clients pick it up on next sync.

**B) Segment fan-out (used by `segment`)** — `sme-user.service.ts → resolveSegment()`
- No topics. A live DB query selects the cohort (see §3 table), then per-user send.

⚠️ **They disagree on trial:** the `tier_premium` *topic* includes trial users; the
`premium` *segment* does not. **For accurate premium/trial/free targeting, use the
`segment` endpoint (§3).** There is currently **no SME endpoint** that pushes the
`tier_premium`/`tier_free` topics directly — `broadcast` only hits `all_users`.

---

## 6. Caveats checklist

- **Permission + dashboard:** topic sends (`broadcast`) only reach devices that opened
  the dashboard with permission granted. Per-user (`/notify`) and `segment` sends reach
  any device with a registered active token (also requires permission).
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
3. **Everyone:** `POST /sme/notifications/broadcast`.
4. **Cohort:** `POST /sme/notifications/segment` with `segment: premium|trial|free`.
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
| Everyone | `POST /sme/notifications/broadcast` | FCM `all_users` topic | `{messageId}` |
| Premium / trial / free | `POST /sme/notifications/segment` | live DB query | `{segment, matchedUsers, successCount, failureCount}` |

All require `x-api-key: <API_KEY_SECRET>`. Prod base: `https://app.stanzasoft.ai/api/v1`.
