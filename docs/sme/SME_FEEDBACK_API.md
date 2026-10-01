# SME Feedback API

> Changed 2026-09-21 — stale-text pass (telemetry live in 2.0, version ladder, links). Exam-dimension changes follow in [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md).
> Changed 2026-09-21 — per-exam broadcast, validated segment exam (+ fix), survey targetExams, feedback ?exam= (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

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
`YYYY-MM-DD`, inclusive), **`exam`**, `page`, `limit` (default 20, max 100).

| `exam` | selects |
|---|---|
| a slug, e.g. `appsc-group-1` | reports **filed from** that exam mode (`feedback_reports.exam_id`, stamped from `req.exam` at submit) |
| `none` | the rows whose `exam_id` is **NULL** — everything filed before 2026-09-21 |
| omitted | everything |

- **Unknown slug → 400** `Unknown exam "<slug>". Create it via POST /sme/exams first.`
  (The column has no FK, so a typo would otherwise return an empty page that reads as
  "this exam has no complaints".)
- `tier` remains **person-level** (the `user_auth` premium mirror), not per-exam. The two
  filters compose but answer different questions.

> 🔴 **NULL here does NOT mean `upsc-cse` — this is the opposite of the `active_exam_id`
> rule used everywhere else.** `feedback_reports.exam_id` is stamped going *forward*
> only and historical rows were deliberately not backfilled, so a NULL means **"we did
> not record it"**, not "UPSC". Consequences to build around:
> - `?exam=upsc-cse` **excludes** every pre-2026-09-21 report.
> - `?exam=none` selects exactly that unrecorded history, and nothing else.
> - The per-exam counts therefore start at the deploy and climb; they are a **floor**,
>   not a share of all time. Do not draw the cliff without saying so.

**Response** (raw — this whole object is the body; the `data` key is this endpoint's own
pagination wrapper, **not** a global envelope):

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/feedback/reports?limit=2` → **200**, verbatim. `GET
/sme/feedback/reports?exam=none&limit=1` returns the same body:

```json
{ "data": [], "total": 0, "page": 1, "limit": 2, "hasMore": false }
```

> 🔎 **Nobody has ever filed a report on staging** (`total: 0` both with and without
> `?exam=`), so no populated row could be captured. The **envelope** above is live —
> `{ data, total, page, limit, hasMore }`, the same one `/sme/users` and `/sme/orders`
> use, and *not* the `{ data, meta }` envelope of `/sme/content/*`. The row shape below
> is verified against the DTO.

**Row shape** (from code):
<!-- shape verified against code @c0a8fe5 — staging holds no reports, see note above -->
```jsonc
{ "id": "…", "userId": "…", "type": "ISSUE", "status": "OPEN",
  "categoryKey": "…", "text": "…", "contextType": "…", "contextId": "…",
  "platform": "ios", "appVersion": "1.7", "userTier": "trial",
  "examId": "appsc-group-1",   // NEW — nullable; null = filed before 2026-09-21
  "replyCount": 0, "createdAt": "…" }
```

### Detail — `GET /sme/feedback/reports/:id`

Returns the full report (including `examId`) + `replies` (ascending) + a resolved
**context**:

<!-- shape verified against code @c0a8fe5 — staging holds no reports, so no detail read was possible -->
```jsonc
{
  "examId": "appsc-group-1",          // which exam the USER was in when they filed it
  "context": {
    "resolved": true, "kind": "PYQ_QUESTION", "snippet": "…", "subject": "Polity",
    "examIds": ["*"]                  // which exams the CONTENT belongs to
  }
}
```

**`context.examIds` is a different axis from `examId`** and the two can legitimately
disagree — an APPSC aspirant reporting a `*`-tagged reel gives
`examId: "appsc-group-1"` with `examIds: ["*"]`. Treat a mismatch as information, not a
bug.

`context.examIds` is **omitted, not `[]`**, when there are no tags to report. Read it
with a presence check, never `.length`:

| `contextType` | `context.examIds` |
|---|---|
| `PYQ_QUESTION`, `MAINS_QUESTION`, `CONTENT_DOC`, `REEL` | the content row's tags, e.g. `["appsc-group-1"]` or `["*"]` |
| `SIMULATION_QUESTION` | **absent** — `simulation_questions` carries no tags of its own; the exam lives on the parent simulation. Read it from `GET /sme/content/simulations/:id` if you need it |
| `APP_ERROR` | absent — resolves to `{ resolved: true, kind: "app_error" }` and nothing else |
| unresolved / deleted content | absent (`{ resolved: false }`) |

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
(`userTier`) · "Is it a build-specific problem?" (`appVersion` + `platform`) · "Is this exam
launch going badly?" (`?exam=` + `examId`) · "How long has this person been waiting?"
(`createdAt` + `replyCount`).

An exam filter chip is worth having, but give it a **"not recorded"** option wired to
`?exam=none` and keep it out of any all-time percentage: those rows are not UPSC, they are
unknown, and a per-exam count that silently drops them will read as a sudden surge of
complaints on deploy day.

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

`GET /sme/feedback/reports` → **`total: 0`**. The inbox was genuinely empty at that
measurement: the module shipped on 2026-07-22 and no released build submitted reports yet. App
**2.0** (released 2026-09-17) is the first build expected to, so re-measure before repeating
the zero — and even once it fills, volume tracks 2.0 adoption, since older builds submit
nothing. A low count is "partly measured", not "no complaints". Design the empty state as a
real state with that sentence in it, and make sure the filter chips and pagination degrade
gracefully at zero rows.

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
also means `byRating` was, at the time of that measurement, the only usable field on this
endpoint.

**On `topChips`:** the backend write path is **live and always has been** —
`ChatFeedbackService.submit` persists `dto.chips` straight onto
`chat_message_feedback.chips` before it forwards anything to Dify, so `topChips` is fed by our
own table and does **not** depend on the Dify forward succeeding. Whether it fills up is purely
a question of the client sending a non-empty `chips[]` on a thumbs-down. That is **expected from
app 2.0** (released 2026-09-17) and cannot be proven from the backend — **verify in the data**:
`SELECT count(*) FROM chat_message_feedback WHERE cardinality(chips) > 0;`. Until that returns
non-zero, treat an empty `topChips` as unconfirmed client coverage, not as a broken endpoint.

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
  "targetExams": ["appsc-group-1"],
  "priority": 10, "startAt": "2026-08-01T00:00:00Z", "endAt": "2026-08-31T00:00:00Z"
}
```

Created `DRAFT`. **Question ids are assigned server-side** (any id you send is
ignored) — the returned `questions` carry the stable ids used in results/CSV.
Empty `target*` arrays = everyone.

<!-- captured from staging 2026-09-21, backend f6329e6 -->
**Response `201`**, verbatim — the draft created on staging for the captures in §4
(`targetExams: ["appsc-group-1"]` was the only `target*` array sent):

```json
{
  "id": "6c16b3a6-5b2d-4755-8340-625c098af358",
  "title": "SME-HANDOFF-SMOKE — docs capture",
  "description": "Draft created 2026-09-21 to capture response shapes for the SME docs handoff. Never activated. Safe to delete.",
  "status": "DRAFT",
  "questions": [
    {
      "id": "684c694d-c2e1-4be3-99c0-830bf2aaa25f",
      "text": "Smoke question?",
      "type": "single_choice",
      "options": [{ "key": "a", "label": "A" }, { "key": "b", "label": "B" }]
    },
    {
      "id": "37415e64-f4aa-40e6-bb10-cf78c1f57384",
      "text": "How likely are you to recommend PrepMonkey?",
      "type": "nps_0_10"
    }
  ],
  "targetTiers": [],
  "targetPlatforms": [],
  "targetAspirantTypes": [],
  "targetExams": ["appsc-group-1"],
  "priority": 1,
  "startAt": null,
  "endAt": null,
  "activatedAt": null,
  "closedAt": null,
  "createdAt": "2026-09-21T14:56:31.531Z",
  "updatedAt": "2026-09-21T14:56:31.531Z"
}
```

> **The three `target*` arrays that were never sent come back as `[]`, not `null` or
> absent** — so "everyone" is an empty array on read as well as on write, and the portal
> can bind a multi-select straight to them. A non-choice question (`nps_0_10`) carries
> **no `options` key at all** rather than an empty array.

That same survey on `GET /sme/surveys`, verbatim:

```json
[{ "id": "6c16b3a6-5b2d-4755-8340-625c098af358",
   "title": "SME-HANDOFF-SMOKE — docs capture",
   "status": "DRAFT", "priority": 1, "questionCount": 2, "responseCount": 0,
   "activatedAt": null, "createdAt": "2026-09-21T14:56:31.531Z" }]
```

**`targetExams`** is the exam cohort, and behaves exactly like the other `target*`
arrays — **empty or omitted = every exam**, which is what every survey authored before
2026-09-21 carries, so none of them changed audience.

- Every slug is **checked against the exam catalogue on write**: unknown → **400**
  `Unknown exam "<slug>". Create it via POST /sme/exams first.` Slugs are trimmed,
  lower-cased and de-duplicated before storage.
- **`"*"` is rejected** with a **400**: `targetExams does not accept "*" — leave the
  array empty to target every exam.` An empty array already means everyone, and a second
  spelling of it would have to be handled at every read site forever. (This differs from
  content `examIds`, where `*` *is* the sentinel — do not carry that habit over.)
- ⚠️ **`upsc-cse` in the list also matches every user who has never used the exam picker
  or the home switcher** (`active_exam_id` NULL, or no profile row). Without that, a
  UPSC-targeted survey would reach only the minority who have used the switcher.
- `targetTiers` stays **person-level** (the `user_auth` premium mirror): `premium` means
  holds premium *somewhere*, not in the targeted exam.

### List / detail

`GET /sme/surveys` → `[{ id, title, status, priority, questionCount,
responseCount, activatedAt, createdAt }]`.
`GET /sme/surveys/:id` → the full survey + `counts: { answered, snoozed,
dismissed }`.

### Edit — `PATCH /sme/surveys/:id`

`title`, `description`, `priority`, `endAt`, and the `target*` arrays — `targetExams`
included — are editable at any time. **`questions` are structurally frozen once a survey
is `ACTIVE`** (`SurveyStructuralEditBlocked`, 409) — fix by cloning into a new
survey. Questions are editable while `DRAFT`.

`targetExams` on `PATCH` **replaces the whole list** (same validation as create: known
slugs only, no `"*"`). Send `[]` to widen the survey back to every exam; **omit the key**
to leave the current list alone. Changing it re-scopes who is *offered* the survey from
that moment on — it does not retract it from anyone who already answered, and it does not
re-attribute existing responses.

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

**Both now honour `targetExams`** (added 2026-09-21). A survey whose `targetExams` is
non-empty is offered only when the request's exam is in that list; the request's exam is
`X-Exam` / `?exam=`, and **an app that sends neither is treated as `upsc-cse`**, so older
builds keep seeing UPSC-targeted surveys. An empty `targetExams` means every exam, so the
response shape and the set of surveys returned are unchanged for all existing data.

⚠️ **Submit and dismiss are deliberately NOT exam-filtered.**
`POST /feedback/surveys/:id/responses` and `…/dismiss` re-check eligibility (so a client
cannot post to a survey it was never offered) but skip the exam test. The exam on a
request is just the mode the app happens to be showing; a user who was legitimately
offered a survey and then tapped the exam switcher must still be able to answer or
dismiss it. The exam filter governs what gets **offered**, nothing else.

### Announce — `POST /sme/surveys/:id/notify`

Body `{ title, body }`. Fans out a `survey` deep-link to the survey's cohort,
**excluding users who already answered or permanently dismissed it**. Batched
per 50. Returns `{ surveyId, matchedUsers, successCount, failureCount }`.
Audited.

**The cohort is the survey's own `target*` filters, `targetExams` included** (since
2026-09-21) — the push audience and the in-app eligibility now agree. A survey authored
before that field existed has `targetExams: []` and notifies exactly who it always did.
`upsc-cse` in the list also matches users with no stored exam preference.

### Results — `GET /sme/surveys/:id/results?breakdown=tier|platform|exam`

Per question:
- choice → `options: [{ key, label, count, pct }]`
- `rating_1_5` → `average` + `distribution` (1–5)
- `nps_0_10` → `npsScore` (promoters% − detractors%), `promoters/passives/detractors`, `distribution` (0–10)
- `free_text` → `texts` (capped at 500)

`breakdown` adds a per-group `breakdown.groups` for non-text questions.

**`breakdown=exam`** (new) groups by the respondent's exam, and the response gains three
keys **only for this breakdown** — a `tier`/`platform` response is byte-identical to
before:

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/surveys/6c16b3a6-5b2d-4755-8340-625c098af358/results?breakdown=exam` —
verbatim. The survey is a two-question DRAFT created on staging for this capture
(`SME-HANDOFF-SMOKE — docs capture`, never activated) and has **zero responses**, so this
is the **empty-state** body for both question types at once:

```json
{
  "surveyId": "6c16b3a6-5b2d-4755-8340-625c098af358",
  "totalResponses": 0,
  "results": [
    {
      "question": {
        "id": "684c694d-c2e1-4be3-99c0-830bf2aaa25f",
        "text": "Smoke question?",
        "type": "single_choice",
        "options": [{ "key": "a", "label": "A" }, { "key": "b", "label": "B" }]
      },
      "type": "single_choice",
      "answered": 0,
      "options": [
        { "key": "a", "label": "A", "count": 0, "pct": 0 },
        { "key": "b", "label": "B", "count": 0, "pct": 0 }
      ],
      "breakdown": { "by": "exam", "groups": {} }
    },
    {
      "question": {
        "id": "37415e64-f4aa-40e6-bb10-cf78c1f57384",
        "text": "How likely are you to recommend PrepMonkey?",
        "type": "nps_0_10"
      },
      "type": "nps_0_10",
      "answered": 0,
      "npsScore": 0,
      "promoters": 0,
      "passives": 0,
      "detractors": 0,
      "distribution": { "0": 0, "1": 0, "2": 0, "3": 0, "4": 0, "5": 0, "6": 0, "7": 0, "8": 0, "9": 0, "10": 0 },
      "breakdown": { "by": "exam", "groups": {} }
    }
  ],
  "examNullCount": 0,
  "examNullBucketedAs": "upsc-cse",
  "examNote": "0 of 0 respondents have no stored exam preference and are counted under \"upsc-cse\", which is the exam they were served. Exam is read from the profile as it stands today, not snapshotted at answer time, so a respondent who has since switched exams is counted under their current one."
}
```

> **Three empty-state traps that capture exposes:**
>
> * **`breakdown.groups` is `{}`, not absent and not `null`.** Iterate its keys; do not
>   test for the key's presence.
> * **`npsScore: 0` on zero responses.** Unlike `conversionPct` in the analytics API,
>   this one does **not** use `null` for "no denominator" — a brand-new survey scores
>   the same as a genuinely neutral one. **Gate the NPS tile on `answered > 0`**, never
>   on `npsScore`.
> * **`examNote` is still populated at zero** ("0 of 0 respondents…"). It is ready-made
>   copy, not a signal that anything happened; render it beside the chart regardless.
>
> Each result entry carries the **full `question` object** (id, text, type and, for
> choice questions, `options`) alongside a top-level `type` — so the results payload is
> self-describing and the portal need not join back to the survey detail.

> ⚠️ **Exam on a response is RETRO-ATTRIBUTED, not a snapshot.** `survey_responses`
> stores `userTier` and `platform` as they were *at submit time*, but it stores **no
> exam**. `breakdown=exam` therefore reads the respondent's **current**
> `user_profiles.active_exam_id`: somebody who switched exams after answering is counted
> under the exam they are on **today**, and the numbers can move between two reads of the
> same closed survey. A NULL / missing profile is bucketed as `upsc-cse` (the exam that
> user was actually served) and **`examNullCount` is how many rows that was** — surface
> it, or the UPSC bar is an unfalsifiable majority. `examNote` is ready-made copy for
> that caveat.

### Raw responses / CSV

`GET /sme/surveys/:id/responses?page&limit` — paginated raw answered rows. **Every row
now carries `exam`**, with the same retro-attribution as above (current profile value,
NULL → `upsc-cse` — here folded in silently, there is no per-row "unknown" marker).

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/surveys/6c16b3a6-5b2d-4755-8340-625c098af358/responses?page=1&limit=50` →
**200**, verbatim (the smoke survey has no responses):

```json
{ "data": [], "total": 0, "page": 1, "limit": 50 }
```

> **Note the envelope: `{ data, total, page, limit }` with no `hasMore`** — a third
> pagination contract, different from both `/sme/users` (`hasMore`) and `/sme/content/*`
> (`meta`). Compute "is there another page" from `page * limit < total`.

**Row shape** (from code — staging has no survey responses to capture):
<!-- shape verified against code @c0a8fe5 — no responses exist on staging -->
```jsonc
{ "id": "…", "userId": "…", "userTier": "trial", "platform": "ios",
  "aspirantType": "FULL_TIME", "answeredAt": "…", "answers": [],
  "exam": "appsc-group-1" }
```

`GET /sme/surveys/:id/responses/export.csv` — RFC-4180 CSV (UTF-8 + BOM), one
column per question, capped at 50k rows. This is a **raw file download** (no
envelope). It gains **`exam` as the LAST column**, after the question columns —
appended deliberately so no existing column shifts for a sheet or script that reads by
position:

```
responseId,userId,userTier,platform,aspirantType,answeredAt,<question 1>,…,<question n>,exam
```

<!-- captured from staging 2026-09-21, backend f6329e6 -->
The header line of a real export, verbatim
(`GET /sme/surveys/6c16b3a6-5b2d-4755-8340-625c098af358/responses/export.csv`; the two
question columns are the smoke survey's own question **text**, and the file begins with a
UTF-8 BOM):

```
responseId,userId,userTier,platform,aspirantType,answeredAt,Smoke question?,How likely are you to recommend PrepMonkey?,exam
```

> **The question columns are headed by the question text, not by question id** — so a
> script must not key off them, and a survey whose question text contains a comma or a
> quote relies on RFC-4180 quoting to stay parseable. `exam` is confirmed **last**.

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
panel (`targetTiers` / `targetPlatforms` / `targetAspirantTypes` / `targetExams`, where
**empty means everyone**; say that in the field, because an empty multi-select reads as
"nobody"). Populate the exam control from `GET /sme/exams` rather than free text — unknown
slugs and `"*"` are both 400s — and note beside it that UPSC also captures everyone who has
never picked an exam. The
DRAFT → ACTIVE step must be an explicit, deliberate confirmation that states the consequence:
*questions can no longer be edited*. That is enforced server-side with a 409
(`SurveyStructuralEditBlocked`) and the fix is cloning into a new survey, which is expensive
to discover by accident.
*Results:* per question, the shape the data already has — choice questions as ranked bars with
`pct`, `rating_1_5` as a distribution with the `average` called out, `nps_0_10` as the
standard promoters/passives/detractors split with `npsScore` as the headline, free text as a
readable list (capped at 500, so say when it is truncated). `breakdown=tier|platform|exam`
turns each of those into small multiples — offer it as a toggle, not a separate page. On the
`exam` toggle, render `examNote` (or your own wording of it) next to the chart: the numbers
are retro-attributed from today's profile and `examNullCount` of them are in the `upsc-cse`
bucket only because nothing was recorded. Do not let an operator screenshot that bar without
the caveat attached.

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
- **Volume was near zero when this was written.** As of 2026-07-23: 0 reports, 1 chat rating
  (which failed to forward), 3 smoke-test surveys with 2 responses between them. Those counts
  are months old — re-measure before quoting them. Report and chip volume comes from the app,
  and app **2.0** (released 2026-09-17) is the first build expected to supply it, so coverage
  grows with adoption rather than arriving all at once. Build the empty states as first-class
  states — an empty screen that says nothing will be reported as a broken integration.

---

Related: [SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) (the `sendToUser` fan-out and
the deep-link `type` contract used in §5) ·
[SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) (`feedback_reports`,
`survey_responses` and `chat_message_feedback` also appear on a user's timeline) ·
[archive/WHAT_CHANGED_2026-07-23.md](./archive/WHAT_CHANGED_2026-07-23.md) (archived)
