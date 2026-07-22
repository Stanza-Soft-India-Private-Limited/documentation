# SME Feedback API

Server-to-server management of the in-app **feedback** module: user reports
(raise-an-issue / request-a-feature), AI-chat message feedback, SME-authored
surveys, and the tunable chip-set / snooze config.

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Auth:** `x-api-key: <API_KEY_SECRET>` on every `/sme/*` request (`@SmeApiKey`).
All responses ride the global `{ success, message, data, timestamp, path }`
envelope — read `data`. The one exception is the CSV export (raw file download).

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

`data`: `{ data: [ { id, userId, type, status, categoryKey, text, contextType,
contextId, platform, appVersion, userTier, replyCount, createdAt } ], total,
page, limit, hasMore }`.

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

---

## 2. Chat feedback (aggregate)

### Summary — `GET /sme/feedback/chat/summary`

`{ total, byRating: { up, down }, byMode: [{ mode, count }], topChips: [{ key,
count }], difyForwardFailures }`. `difyForwardFailures` counts rows whose Dify
forward did not succeed (`difyForwarded=false` with a recorded `difyError`, e.g.
`no_api_key`).

### Messages — `GET /sme/feedback/chat/messages`

Query: `rating` (default `down`), `mode` (mentor|sme), `page`, `limit`. Returns
the raw rows (messageId, mode, rating, subject, chips, text, difyForwarded,
difyError, createdAt).

---

## 3. Config — `GET / PUT /sme/feedback/config`

`GET` returns the **effective** config (defaults overlaid by any stored
overrides):

```json
{
  "chipSets": { "chat": { "chips": [{ "key": "inaccurate", "label": "Inaccurate answer" }] }, "…": {} },
  "survey": { "snoozeDays": 3, "maxDismissals": 2 },
  "limits": { "reportsPerDay": 10, "chatPerDay": 30 }
}
```

The 9 chip sets are `chat`, `pyq_question`, `mains_question`,
`simulation_question`, `content_doc`, `reel`, `general_issue`,
`feature_request`, `app_error`.

`PUT` accepts a partial `{ chipSets?, survey?, limits? }`. Only the sections you
send are touched (row-locked read-modify-write). Chip **keys** are stable ids —
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
- **Dify forwarding** is out-of-band and best-effort. A missing per-subject key
  (`DIFY_APP_KEYS`) shows up as `difyError=no_api_key` in the chat summary — it
  never fails the user's thumb.
- **Tier** everywhere is PostgreSQL-derived, not the graph.
- **IST** — the report list `from`/`to` are IST calendar dates.
