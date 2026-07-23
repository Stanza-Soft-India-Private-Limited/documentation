# SME Feedback API

> ## ⛔ BEFORE YOU WRITE ANY UI CODE — INVOKE THE `frontend-design` SKILL
> If you are a Claude/coding session building screens on this API, your **first action** is
> to invoke the **`frontend-design`** skill (`/frontend-design`). Do this **even if no one
> asked you to** and even if the surrounding task didn't mention design. Every queue row,
> reply thread, survey editor field and empty state below must be designed through that
> skill — do not reach for a default admin CRUD scaffold. This is a hard requirement, not a
> suggestion.
>
> This doc produces **three genuinely different screens**, and collapsing them into one
> "Feedback" tab is the failure mode to avoid:
>
> 1. **The report inbox — a triage queue with a human on the other end.** Someone typed a
>    complaint into the app and is waiting to hear back. It is read by support daily, and
>    every row is either "answer this" or "close this". Unread-and-old is the state the
>    design must make uncomfortable. There is no user-side reply, so a reply you send is
>    final — the compose box should feel like sending, not chatting.
> 2. **Chat feedback — an analytics read, not a queue.** Nobody is waiting. There is no
>    reply path at all. Design it as a small summary panel, and resist giving it the same
>    row-with-actions treatment as the inbox; an operator who thinks they can respond to a
>    thumbs-down will waste time looking for the button.
> 3. **The survey authoring + results tool — a two-mode editor.** Authoring is a
>    forms-heavy composer used rarely and carefully; results are a read-only report used
>    after the fact. They share a survey and nothing else. Question structure **freezes on
>    activation** (409 thereafter), so the editor has to make the DRAFT → ACTIVE transition
>    feel irreversible *before* it is taken, not explain it in an error toast afterwards.
>
> Audience for all three = internal ops/content staff, not consumers.
>
> Paste this with the doc when you brief an agent:
>
> ```
> Invoke the /frontend-design skill first, before writing any UI code.
> Then build the SME feedback surfaces per SME_FEEDBACK_API.md.
> Three distinct screens, not one tab:
>   report inbox   = a triage QUEUE (a real user is waiting; replies are one-way and final)
>   chat feedback  = a read-only ANALYTICS panel (no reply path exists — do not imply one)
>   surveys        = an AUTHORING tool + a results report; questions FREEZE on activate (409)
> Responses are RAW JSON — there is NO {success,data} envelope. Branch on HTTP status.
> List endpoints return a per-endpoint pagination wrapper {data,total,page,limit,hasMore};
> that is a documented per-endpoint shape, not a global envelope. The CSV is a file download.
> Design the empty states first — the report inbox is currently empty in production.
> ```

Server-to-server management of the in-app **feedback** module: user reports
(raise-an-issue / request-a-feature), AI-chat message feedback, SME-authored
surveys, and the tunable chip-set / snooze config.

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Auth:** `x-api-key: <API_KEY_SECRET>` on every `/sme/*` request (`@SmeApiKey`).

> ### ⚠️ CORRECTION (2026-07-23) — there is NO response envelope
> An earlier version of this document stated that all responses ride a global
> `{ success, message, data, timestamp, path }` envelope and that you should read
> `data`. **That was wrong**, and it is the single most expensive mistake in this
> codebase's history: a mobile release built envelope-decoding on top of it, the
> surveys inbox crashed on-device (`[` where `{` was expected) and the survey card
> silently rendered nothing.
>
> The truth, verified against the registration line rather than inferred:
> `src/common/interceptors/response.interceptor.ts` defines such a wrapper but is
> **registered nowhere**. The sole global `APP_INTERCEPTOR` in `app.module.ts` is
> `ApiUsageInterceptor`. **Success bodies go out RAW** — a list endpoint returns a
> bare JSON array, an object endpoint returns the bare object.
>
> Error bodies *are* shaped, but by `exception.filter.ts`, not by an envelope:
> `{ success: false, message, error, statusCode, timestamp, path, method }`.
>
> **Client rule: branch on the HTTP status code, never on the presence of `success`.**
> A few endpoints wrap manually inside their own controller — trust the per-endpoint
> response shape documented below, not a global assumption.

The CSV export returns a raw file download.

---

## 0. The mental model

Four channels land here, all authenticated on the app side (userId + tier +
platform are attached server-side for segmentation):

1. **Reports** — a category chip + free text, typed `ISSUE` or `FEATURE`,
   optionally pre-tagged to a screen (a PYQ/Mains/Simulation question, a study
   doc, a reel, or an app error). You triage them: change status, reply.
2. **Chat feedback** — thumbs up/down on an AI answer, forwarded to Dify. You get
   an aggregate view (counts, top chips, forward failures), not a triage queue.
3. **Surveys** — you author them (paginated question runner in-app), target a
   cohort, activate, push, and read per-question results (+ CSV).
4. **Config** — the chip sets shown in the forms and the survey snooze policy,
   tunable without an app release.

Tier is resolved from **PostgreSQL only** (never the graph): `premium` (active
paid), `trial` (in-window trial), `free` (everyone else).

---

## 1. Reports

### List — `GET /sme/feedback/reports`

Query params (all optional): `type` (ISSUE|FEATURE), `status`, `categoryKey`,
`platform` (ios|android|web), `tier` (premium|trial|free), `contextType`,
`userId`, `search` (substring of the free text), `from` / `to` (IST dates
`YYYY-MM-DD`, inclusive), `page`, `limit` (default 20, max 100).

**Response** (raw — this whole object is the body; the `data` key is this endpoint's own
pagination wrapper, **not** a global envelope):

```jsonc
{
  "data": [ { "id": "…", "userId": "…", "type": "ISSUE", "status": "OPEN",
              "categoryKey": "…", "text": "…", "contextType": "…", "contextId": "…",
              "platform": "ios", "appVersion": "1.7", "userTier": "trial",
              "replyCount": 0, "createdAt": "…" } ],
  "total": 0, "page": 1, "limit": 20, "hasMore": false
}
```

### Detail — `GET /sme/feedback/reports/:id`

Returns the full report + `replies` (ascending) + a resolved **context**:

```json
{ "context": { "resolved": true, "kind": "PYQ_QUESTION", "snippet": "…", "subject": "Polity" } }
```

If the referenced content was deleted (or there was no context), `context` is
`{ "resolved": false }` — never a 500.

### Status — `PATCH /sme/feedback/reports/:id/status`

Body `{ status, actorNote? }`. Statuses: `OPEN → IN_REVIEW → RESOLVED / CLOSED`;
FEATURE reports additionally allow `PLANNED` and `SHIPPED`. **`PLANNED` /
`SHIPPED` on an ISSUE are rejected** (`FeedbackStatusInvalidForType`, 400). The
change is audited and the user is pushed a `report` deep-link (see §5).

### Reply — `POST /sme/feedback/reports/:id/replies`

Body `{ text, actorNote? }`. Users **read** replies (this is not a chat — there
is no user-side reply). Audited + pushes a `report` deep-link. A failed push
never rolls back the reply.

### How to use this data

**Questions it answers.** "What are users complaining about, and about what part of the
product?" (`categoryKey` + `contextType`/`contextId`) · "Is this coming from paying users?"
(`userTier`) · "Is it a build-specific problem?" (`appVersion` + `platform`) · "How long has
this person been waiting?" (`createdAt` + `replyCount`).

**The decision it drives.** Reply now, escalate, or close. Secondarily — and this is the part
teams forget — **which product area to fix**, because `categoryKey` and `contextType`
aggregated over a month is a ranked list of what annoys users, produced for free by the triage
work you were doing anyway.

**What a good visualisation is.** A queue, sorted by **age of the oldest unanswered report**,
not by newest. Each row: the first line of `text` at full readable size (the text is the
content — do not truncate it to 40 characters to fit a tidy column), a status chip, the tier
badge, the category, and elapsed time. Status filters as tabs across the top with counts.
On the detail view, put the resolved `context` block — the actual question or document the
user was looking at — directly beside their complaint; that adjacency is the entire value of
having pre-tagged the report, and it turns "this question is wrong" into something actionable
without leaving the page. Handle `context: { resolved: false }` as a plain "the referenced
content no longer exists" line, not a broken card.

**The action that follows.** `PATCH …/status` then `POST …/replies`. Both push a `report`
deep-link to the user, so the reply is a real customer-facing message — treat the compose box
accordingly. Remember `PLANNED` / `SHIPPED` are FEATURE-only; a 400 `FeedbackStatusInvalidForType`
means the UI offered a status it shouldn't have. Gate the options on `type` instead of
letting the server reject it.

#### What it actually returns today — measured 2026-07-23

`GET /sme/feedback/reports` → **`total: 0`**. The inbox is genuinely empty: the module shipped
on 2026-07-22 and the mobile build that submits reports has not been released. This is "no
data yet", not "no complaints" — and it is the state the screen will be in on the day it is
handed over. Design the empty state as a real state with that sentence in it, and make sure
the filter chips and pagination degrade gracefully at zero rows.

---

## 2. Chat feedback (aggregate)

### Summary — `GET /sme/feedback/chat/summary`

`{ total, byRating: { up, down }, byMode: [{ mode, count }], topChips: [{ key,
count }], difyForwardFailures }`. `difyForwardFailures` counts rows whose Dify
forward did not succeed (`difyForwarded=false` with a recorded `difyError`, e.g.
`no_api_key`).

### Messages — `GET /sme/feedback/chat/messages`

Query: `rating` (default `down`), `mode` (mentor|sme), `page`, `limit`. Returns
the same `{ data, total, page, limit, hasMore }` pagination wrapper as the report
list, with rows of `{ id, userId, messageId, mode, rating, subject, chips, text,
difyForwarded, difyError, createdAt }`.

⚠️ `rating` defaults to **`down`**. A screen that calls this endpoint with no params and shows
"no results" may simply mean there are no thumbs-*down* — pass `rating=up` explicitly to see
the other side, and label which one you are showing.

### How to use this data

**Questions it answers.** "Are people happy with the AI answers?" (crudely — `byRating`) ·
"What do they complain about when they aren't?" (`topChips`) · "Which mode is being used?"
(`byMode`) · **"Is the Dify forward actually working?"** (`difyForwardFailures`).

**The decision it drives.** Two, and the second is the one that matters more often. First,
whether a prompt or a subject app needs work — `topChips` plus the free text on downvotes is
the signal. Second, and it is an **operational** decision rather than a product one:
`difyForwardFailures` is an integration health check hiding in an analytics endpoint. Every
failure means a user's rating never reached Dify, so the model never learned from it.

**What a good visualisation is.** A compact summary block — up/down split, mode split, chip
frequency as a short ranked list — and a separate, visually distinct alert line for
`difyForwardFailures` whenever it is non-zero, because that is an ops problem and not a
product metric. Then the raw rows beneath, defaulting to downvotes, with `text` shown in full.
No reply affordance anywhere; there is no reply path.

#### What it actually returns today — measured 2026-07-23

```jsonc
// GET /sme/feedback/chat/summary
{ "total": 1, "byRating": { "up": 1, "down": 0 }, "byMode": [ { "mode": "MENTOR", "count": 1 } ],
  "topChips": [], "difyForwardFailures": 1 }
```

One rating in the entire production history — and **it failed to forward**:

```jsonc
// GET /sme/feedback/chat/messages?rating=up
{ "data": [ { "mode": "MENTOR", "rating": "UP", "subject": null, "chips": [], "text": null,
              "difyForwarded": false, "difyError": "no_api_key",
              "createdAt": "2026-07-22T04:38:09.600Z" } ], "total": 1 }
```

`difyError: "no_api_key"` means `DIFY_APP_KEYS` has no entry for the `mentor` mode on the
environment that served that request, so the forward short-circuited before it was attempted.
**100% of chat ratings collected so far have failed to reach Dify.** This is a real,
outstanding production configuration gap, not a sample artefact — it is reported here rather
than smoothed over, and it is exactly the case `difyForwardFailures` exists to surface. It
also means `byRating` is currently the only usable field on this endpoint, and `topChips` will
stay empty until the app release that collects chips ships.

---

## 3. Config — `GET / PUT /sme/feedback/config`

`GET` returns the **effective** config (defaults overlaid by any stored
overrides):

```json
{
  "chipSets": { "chat": { "chips": [{ "key": "inaccurate", "label": "Inaccurate answer" }] }, "…": {} },
  "survey": { "snoozeDays": 3, "maxDismissals": 2, "listWindowDays": 30 },
  "limits": { "reportsPerDay": 10, "chatPerDay": 30 }
}
```

The 9 chip sets are `chat`, `pyq_question`, `mains_question`,
`simulation_question`, `content_doc`, `reel`, `general_issue`,
`feature_request`, `app_error`.

`PUT` accepts a partial `{ chipSets?, survey?, limits? }`. Only the sections you
send are touched (row-locked read-modify-write). `survey.listWindowDays` (default
30) bounds how far back the user-facing surveys inbox looks — see §4.1. Chip
**keys** are stable ids —
`^[a-z0-9_]{1,40}$`, unique within a set, each with a non-empty label; **labels**
are freely editable. A write busts the public `GET /feedback/config` cache
immediately and is audited (only the affected sections).

> Change a chip's **label** freely. Do **not** repurpose an existing **key** —
> historical reports/feedback reference it.

---

## 4. Surveys

Question types: `single_choice`, `multi_choice`, `rating_1_5`, `nps_0_10`,
`free_text`. All questions are required in the runner. Options are `{ key, label }`.

### Create — `POST /sme/surveys`

```json
{
  "title": "…", "description": "…",
  "questions": [
    { "type": "single_choice", "text": "…", "options": [{ "key": "a", "label": "A" }] },
    { "type": "nps_0_10", "text": "How likely…" }
  ],
  "targetTiers": ["premium"], "targetPlatforms": ["ios"], "targetAspirantTypes": ["FULL_TIME"],
  "priority": 10, "startAt": "2026-08-01T00:00:00Z", "endAt": "2026-08-31T00:00:00Z"
}
```

Created `DRAFT`. **Question ids are assigned server-side** (any id you send is
ignored) — the returned `questions` carry the stable ids used in results/CSV.
Empty `target*` arrays = everyone.

### List / detail

`GET /sme/surveys` → `[{ id, title, status, priority, questionCount,
responseCount, activatedAt, createdAt }]`.
`GET /sme/surveys/:id` → the full survey + `counts: { answered, snoozed,
dismissed }`.

### Edit — `PATCH /sme/surveys/:id`

`title`, `description`, `priority`, `endAt`, and the `target*` arrays are
editable at any time. **`questions` are structurally frozen once a survey is
`ACTIVE`** (`SurveyStructuralEditBlocked`, 409) — fix by cloning into a new
survey. Questions are editable while `DRAFT`.

### Lifecycle

`POST /sme/surveys/:id/activate` → `ACTIVE` (stamps `activatedAt`).
`POST /sme/surveys/:id/close` → `CLOSED`.

Multiple surveys can be `ACTIVE`; the app shows **at most one** per user
(deterministic: priority desc → activatedAt asc → id). Activation itself is
silent (no push).

### 4.1 User-facing surfaces — dashboard card vs. surveys inbox

The app has two read surfaces over the same eligibility, and they intentionally
disagree on one thing: **dismiss**.

- **`GET /feedback/surveys/current`** — the single dashboard-card pick
  (priority desc → activatedAt asc → id, first eligible). A `SNOOZED` survey is
  excluded until `snoozedUntil` lapses; a `DISMISSED_PERMANENT` survey is
  excluded forever. This is the "don't nag me" surface.
- **`GET /feedback/surveys/open`** — the full surveys inbox (every open survey,
  `activatedAt desc`). Snooze/permanent-dismiss do **not** exclude a survey
  here — dismiss only silences the dashboard card, it does not make a survey
  unreachable. Only an `ANSWERED` response excludes a survey from this list.
  Also bounded by `survey.listWindowDays` (§3): a survey older than that many
  days since `activatedAt` drops out of the inbox even if still eligible and
  unanswered, so the list can't grow unbounded as old surveys pile up.

Both are per-user JWT endpoints (not cached — cohort + response state are
per-user) and return the same survey shape `{ id, title, description,
questions }`; `/open` additionally includes `activatedAt` for inbox sorting.

### Announce — `POST /sme/surveys/:id/notify`

Body `{ title, body }`. Fans out a `survey` deep-link to the survey's cohort,
**excluding users who already answered or permanently dismissed it**. Batched
per 50. Returns `{ surveyId, matchedUsers, successCount, failureCount }`.
Audited.

### Results — `GET /sme/surveys/:id/results?breakdown=tier|platform`

Per question:
- choice → `options: [{ key, label, count, pct }]`
- `rating_1_5` → `average` + `distribution` (1–5)
- `nps_0_10` → `npsScore` (promoters% − detractors%), `promoters/passives/detractors`, `distribution` (0–10)
- `free_text` → `texts` (capped at 500)

`breakdown` adds a per-group `breakdown.groups` for non-text questions.

### Raw responses / CSV

`GET /sme/surveys/:id/responses?page&limit` — paginated raw answered rows.
`GET /sme/surveys/:id/responses/export.csv` — RFC-4180 CSV (UTF-8 + BOM), one
column per question, capped at 50k rows. This is a **raw file download** (no
envelope).

### How to use this data

**Questions it answers.** "What do users think about X?", where X is whatever you asked —
this is the only surface in the product where you get to choose the question. Plus the
operational ones: "did anyone answer?" (`responseCount`), "who ignored it?"
(`counts.snoozed` / `counts.dismissed`).

**The decision it drives.** Authoring side: who to target and when to activate/close.
Results side: whatever the survey was for. The NPS and rating aggregates are pre-computed
(`npsScore`, `distribution`, `average`) so the portal does not re-derive them and drift.

**What a good visualisation is.** Two distinct modes, sharing nothing but the survey id.
*Authoring:* a linear composer — question list with type, text and options — plus a targeting
panel (`targetTiers` / `targetPlatforms` / `targetAspirantTypes`, where **empty means
everyone**; say that in the field, because an empty multi-select reads as "nobody"). The
DRAFT → ACTIVE step must be an explicit, deliberate confirmation that states the consequence:
*questions can no longer be edited*. That is enforced server-side with a 409
(`SurveyStructuralEditBlocked`) and the fix is cloning into a new survey, which is expensive
to discover by accident.
*Results:* per question, the shape the data already has — choice questions as ranked bars with
`pct`, `rating_1_5` as a distribution with the `average` called out, `nps_0_10` as the
standard promoters/passives/detractors split with `npsScore` as the headline, free text as a
readable list (capped at 500, so say when it is truncated). `breakdown=tier|platform` turns
each of those into small multiples — offer it as a toggle, not a separate page.

**The action that follows.** Activate, `POST …/notify` to announce it (the fan-out already
excludes users who answered or permanently dismissed — do not filter again client-side), then
close it and export the CSV.

**The one behaviour worth surfacing in the UI:** the dashboard card and the surveys inbox
deliberately disagree about dismissal (§4.1). A survey a user "dismissed" is gone from the
card forever but still reachable in the inbox until answered or until
`survey.listWindowDays` lapses. If an operator asks "why is this survey still showing for
someone who dismissed it", that is the answer, and it belongs as a note on the survey detail
page rather than in a support thread.

#### What it actually returns today — measured 2026-07-23

`GET /sme/surveys` → three surveys, one in each lifecycle state, all titled *"Help us make
PrepMonkey better"*: `ACTIVE` (5 questions, **2 responses**, activated 2026-07-22),
`CLOSED` (4 questions, 0 responses) and `DRAFT` (4 questions, `activatedAt: null`). They are
the authoring smoke-test from launch day, so the results screens have almost nothing to render
— but the three states are all reachable in production today, which makes this a good surface
to build against. Note `priority: 0` on all three: with a single active survey the
deterministic pick (priority desc → activatedAt asc → id) is never exercised, so test the
tie-break deliberately rather than assuming it works.

---

## 5. Deep-links

Both are sent automatically by the module via `sendToUser` (persists a feed row
+ push). See `SME_NOTIFICATIONS_API.md` §4.

| `type` | `id` | opens | universal link |
|---|---|---|---|
| `report` | reportId | My Reports → that report | `/open/report/<id>` |
| `survey` | surveyId | the survey runner | `/open/survey/<id>` |

---

## 6. Audit

Every mutation writes `sme_audit_log` (best-effort — a failed audit never blocks
the action): `FEEDBACK_REPORT_STATUS`, `FEEDBACK_REPORT_REPLY`,
`FEEDBACK_CONFIG_UPDATE`, `SURVEY_CREATE`, `SURVEY_UPDATE`, `SURVEY_ACTIVATE`,
`SURVEY_CLOSE`, `SURVEY_NOTIFY`.

---

## 7. Caveats

- **Reports are never deleted here** — triage via status (`RESOLVED`/`CLOSED`).
- **Chat feedback is not a queue** — it is analytics; there is no reply path.
- **Dify forwarding** is out-of-band and best-effort. The thumb forward in §2 is the
  only Dify call this backend makes — chat turns, mains evaluation, flashcards and
  practice-similar all go client→Dify direct and never touch us. A missing per-subject
  key (`DIFY_APP_KEYS`) shows up as `difyError=no_api_key` in the chat summary — it
  never fails the user's thumb. ⚠️ **On production today there is no `mentor` key, so
  every forward fails and `difyForwardFailures` equals `total`.** That is a live
  server-side config gap, not a portal bug — see §2.
- **Tier** everywhere is PostgreSQL-derived, not the graph.
- **IST** — the report list `from`/`to` are IST calendar dates.
- **Nothing here is populated at scale yet.** As of 2026-07-23: 0 reports, 1 chat rating
  (which failed to forward), 3 smoke-test surveys with 2 responses between them. The report
  and chip data depend on a mobile release that has not shipped. Build the empty states as
  first-class states — they are what the portal team will see on day one, and an empty
  screen that says nothing will be reported as a broken integration.

---

Related: [SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) (the `sendToUser` fan-out and
the deep-link `type` contract used in §5) ·
[SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) (`feedback_reports`,
`survey_responses` and `chat_message_feedback` also appear on a user's timeline) ·
[WHAT_CHANGED_2026-07-23.md](./WHAT_CHANGED_2026-07-23.md)
