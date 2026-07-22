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
Same body fields as §1 (no `:id` path param; `type`/`id` optional, `type` default `home`).

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
| type / id | ➖ | as above |

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

`type` (+ optional `id`) is forwarded in the FCM **data payload**; when the user taps,
the app routes via `DeepLinkMapper.fromData`. **Only these values route** — anything
else lands on **Home**:

| `type` | needs `id` | opens | universal link |
|---|---|---|---|
| `home` (default) | — | Home / dashboard | `/open/home` |
| `daily_task` | — | Daily tasks (Home) | — |
| `chat` | — | Chat | `/open/chat` |
| `chat_expert` | — | Chat, **Expert (SME) mode** preselected | `/open/chat/expert` |
| `mains` | — | PYQ tab → **Mains** landing | `/open/mains` |
| `mains_question` | ✅ | that Mains question detail | `/open/mains/<id>` |
| `reel` | optional | a specific reel (with `id`) or the Reels feed | `/open/reel[/<id>]` |
| `reelblog` | ✅ | that reel's blog article | `/open/reelblog/<id>` |
| `pyq` | — | PYQ / Practice tab | `/open/pyq` |
| `pyq_question` | ✅ | a specific PYQ question | `/open/pyq/<id>` |
| `simulation` | ✅ | Library → Simulation, highlighted | `/open/simulation/<id>` |
| `doc` | ✅ | a study document | `/open/doc/<id>` |
| `library` | — | Library / My Content | `/open/library` |
| `flashcards` | — | Flashcards | `/open/flashcards` |
| `mnemonics` | — | Mnemonics | `/open/mnemonics` |
| `saved` | — | Saved questions | `/open/saved` |
| `premium` | — | Upgrade / Premium screen | `/open/premium` |
| `report` | ✅ | that feedback report's detail (My Reports) | `/open/report/<id>` |
| `survey` | ✅ | the survey runner for that survey | `/open/survey/<id>` |

Anything not in this table falls back to **Home**. `report` and `survey` are sent
automatically by the feedback module — `report` on a status change or SME reply
(`id` = reportId), `survey` by `POST /sme/surveys/:id/notify` (`id` = surveyId). A
survey deep-link to an expired/replaced survey lands on a graceful "no longer
available" state, never an error screen. `type` is a free string on the send
API (no backend change to use a new one). ⚠️ The `chat_expert`, `mains`, and the parity
types (`pyq`/`doc`/`simulation`/`library`/`premium`/…) route only on **app builds ≥ the
release that shipped them** — older installs fall back to Home for those *push* types
(universal links degrade more gracefully). Adding a brand-new target that isn't an
existing app screen still needs an app release.

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
- **Auditing:** per-user + segment sends are recorded in `sme_audit_log` (`USER_NOTIFY`).

---

## 7. Implementation checklist (SME portal)

1. Store `API_KEY_SECRET` **server-side only** — never ship it to a browser/client.
2. **One user:** find the user via `GET /sme/users?search=...` → take `id` →
   `POST /sme/users/:id/notify`.
3. **Everyone:** `POST /sme/notifications/broadcast`.
4. **Cohort:** `POST /sme/notifications/segment` with `segment: premium|trial|free`.
5. Pick a `type` from §4 (default `home`); pass `id` when the type needs it.
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
