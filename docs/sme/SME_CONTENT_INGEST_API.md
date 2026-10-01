# SME Content Ingest API — `/sme/content/*` (key-gated, exam-safe)

> Changed 2026-09-21 — exam-safe ingest: /sme/content/* + examIds on /cms and content-doc (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).
> Changed 2026-09-21 — per-exam broadcast, validated segment exam (+ fix), survey targetExams, feedback ?exam= (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).
> Changed 2026-10-01 — PYQ `paperLabel` / `questionImageUrl` / `imageCaption` / `boardDeleted`, the answer-key write rule, `409 QUESTION_WITHDRAWN`, and figure upload via `pyq-figures` — **§9**.

The ingest surface the SME portal should call. Same three question banks as `/cms/*`
(PYQ · Mains · Simulations), behind `x-api-key`, with **`examIds` required on every
create** and every write recorded in the audit trail.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `https://app.stanzasoft.ai`)
**Auth:** `x-api-key: <API_KEY_SECRET>` on every request · **Swagger:** `/api/docs` (group **SME**)
**Legacy surface:** [CMS_ADMIN_API.md](./CMS_ADMIN_API.md) — `/cms/*`, still public, still working, not extended.
**Exam catalogue:** [SME_EXAMS_API.md](./SME_EXAMS_API.md) — where slugs come from and how to check a load landed.

> **There is no response envelope.** `ResponseInterceptor` exists in the codebase and is
> registered nowhere, so success bodies are RAW — exactly the shapes below. Errors are the
> Nest default `{ statusCode, message, error }`. Branch on the HTTP status code.

---

## 0. Why this exists

`/cms/*` is `@Public()` and un-keyed (a standing decision — the import scripts depend on
it and it is hidden at the API gateway), its `examIds` is optional, and nothing it does is
audited. That combination has one specific failure mode, and it is the reason this surface
was built:

**A bulk upload for APPSC that forgets `examIds` succeeds, returns `201`, and lands every
row in the UPSC catalogue.** Nothing errors. The APPSC learner sees an empty screen, the
UPSC learner sees questions about Andhra Pradesh, and the only way to find out is
`GET /sme/exams/:id/content-counts` some days later.

`/sme/content/*` closes that hole at the door: no `examIds`, no write. It is **the same
three services** behind the same writes — `CmsPyqService`, `CmsMainsService`,
`CmsSimulationService` — so normalisation, the `isOptional` derivation and the merge-on-PATCH
rule cannot drift between the two surfaces. The only things added here are the
required-`examIds` gate and the audit row.

**Use `/sme/content/*` for every new integration.** Existing import scripts can stay on
`/cms/*`; §0 of [CMS_ADMIN_API.md](./CMS_ADMIN_API.md) is the route-by-route migration map.

---

## 1. Auth

`@SmeApiKey()` on the controller — the same gate as every other `/sme/*` route. It applies
`@Public()` (bypassing the global JWT guard) plus the shared-secret `ApiKeyGuard`.

```bash
curl -X POST {{BASE_URL}}/api/v1/sme/content/pyq \
  -H "x-api-key: $API_KEY_SECRET" \
  -H "Content-Type: application/json" \
  -d '{ … }'
```

A missing or wrong key is a `401`. There is no per-user session here and no actor id —
which is why the audit row records the route and the payload shape rather than a person.

---

## 2. The exam rules

### 2.1 `examIds` is required on every create

Every `POST` that creates a row — `pyq`, `pyq/bulk` (per item), `mains`, `mains/bulk` (per
item), `simulations` — must carry a non-empty `examIds` array. There is no fallback.

```jsonc
"examIds": ["appsc-group-1"]        // this exam only
"examIds": ["upsc-cse", "tpsc-ccs"] // two exams
"examIds": ["*"]                    // EVERY exam, including ones that do not exist yet
```

**The `*` sentinel** means "shared by every exam". It exists so that content which is
genuinely identical across exams — current affairs, core GS material — does not have to be
re-tagged every time a new exam launches. It is stored literally, and every read filters on
`examIds hasSome [<exam>, "*"]`, so `*`-tagged rows are never hidden from anyone. `*` is
**exempt from the catalogue check**: it is not a slug and is never looked up.

### 2.2 Normalisation

Every other value goes through the exam catalogue, which **trims and lower-cases** before
looking it up, then **de-duplicates** the result:

| Sent | Stored |
|---|---|
| `[" APPSC-Group-1 "]` | `["appsc-group-1"]` |
| `["upsc-cse", "UPSC-CSE"]` | `["upsc-cse"]` |
| `["*", "*"]` | `["*"]` |
| `["appsc-grp-1"]` (well-formed, not a real exam) | *nothing — 400* |
| `["APPSC Group 1"]` (not a valid slug shape) | *nothing — 400* |

This is the half that used to bite silently: `/cms/*` stored `" APPSC-Group-1 "` **as that
literal string**, which matched no filter and made the content invisible in its own
catalogue with no error anywhere. That padding case is now normalised on `/cms/*` too, and
since 2026-09-21 `/cms/*` and content-doc admin also **consult the catalogue and log a WARN**
for a well-formed slug that names no exam.

But a WARN is not a refusal: `/cms/*` still **stores `appsc-grp-1` verbatim and returns
`201`**, and nothing in the response says so — you have to be reading the server log at the
time. **Only this surface looks the slug up and refuses**, which is the difference between
catching the typo now and finding it in `content-counts` a week later.

### 2.3 The two 400s, verbatim

<!-- captured from staging 2026-09-21, backend f6329e6 -->
Missing, `null`, or `[]` — `POST /sme/content/pyq` with an otherwise-valid body and no
`examIds` (nothing was written):

```json
{
  "success": false,
  "message": "examIds is required: list the exam slugs this content belongs to, or [\"*\"] for every exam.",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:52:54.647Z",
  "path": "/api/v1/sme/content/pyq",
  "method": "POST"
}
```

A slug that is not in the exam catalogue (`<slug>` is echoed exactly as you sent it, before
trimming) — captured on `GET /sme/banners?exam=appsc-grp-1`, which shares this validator:

```json
{
  "success": false,
  "message": "Unknown exam \"appsc-grp-1\". Create it via POST /sme/exams first.",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:37:40.351Z",
  "path": "/api/v1/sme/banners?exam=appsc-grp-1",
  "method": "GET"
}
```

> **Corrected 2026-09-21:** earlier revisions of this doc showed these as a three-key
> `{ statusCode, message, error }` body. The real envelope has **seven** keys, in the
> order above — `success` first, `statusCode` *after* `error`. Match on `message`.

⚠️ **`message` is a single string on both**, not the array a `ValidationPipe` failure
produces. That is deliberate — the check is raised by the controller, not by
class-validator, precisely so the portal can match on one sentence. Do not write a handler
that assumes `message` is an array.

### 2.4 `PATCH` merges — but an empty list is still an error

| Body | Effect on `examIds` |
|---|---|
| key omitted | **untouched** — the row keeps its current tags |
| `{"examIds": ["appsc-group-1"]}` | validated, normalised, replaces the column |
| `{"examIds": []}` | **400** (`examIds is required: …`) |
| `{"examIds": ["nope"]}` | **400** (unknown exam) |

An omitted key must never be re-defaulted: re-tagging an APPSC paper as UPSC because
someone fixed a typo is the same bug in a different coat. An *explicitly empty* list, on
the other hand, cannot mean "leave it alone" — it is refused rather than guessed at.

> ⚠️ This is the one place `/sme/content/*` and `/cms/*` behave differently on the same
> body: `PATCH /cms/pyq/:id {"examIds": []}` resolves to `["upsc-cse"]` and silently
> re-tags the row.

### 2.5 Everything is audited

Every write records one `sme_audit_log` row with action **`CONTENT_INGEST`**,
`targetUserId: null`, and an `after` payload:

<!-- captured from staging 2026-09-21, backend f6329e6 -->
The row below is the **actual `sme_audit_log` row** written by the `POST
/sme/content/pyq` capture in §3.1, read straight out of staging Postgres on 2026-09-21
(`SELECT action, target_user_id, after, created_at FROM sme_audit_log WHERE
action='CONTENT_INGEST' ORDER BY created_at DESC LIMIT 1`):

```json
{
  "action": "CONTENT_INGEST",
  "target_user_id": null,
  "after": {
    "route": "POST /sme/content/pyq",
    "count": 1,
    "examIds": ["appsc-group-1"],
    "id": "94534edf-ffe3-4b4a-a531-a7e438abdb10"
  },
  "created_at": "2026-09-21T14:49:51.368"
}
```

A bulk route writes the same `after` **without** the `id` key:

```jsonc
{ "route": "POST /sme/content/pyq/bulk", "count": 100, "examIds": ["appsc-group-1"] }
```

> ℹ️ **There is no API to read this back.** `sme_audit_log` is exposed only through the
> per-user timeline ([SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md)), and
> `CONTENT_INGEST` rows carry `targetUserId: null` — so they belong to no user's timeline
> and are invisible to every `/sme/*` read. Capturing the row above required a direct
> database query. **If the portal needs a content-ingest history screen, that endpoint
> does not exist yet.**

One verb covers create, bulk and update across all three banks rather than nine — the thing
worth reconstructing afterwards is *what landed in which exam*, and `route` + `count` +
`examIds` says exactly that. Single-row routes add `{ "id": … }`; the two simulation-question
routes carry `examIds: null` (§5). Read the trail through
[SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md).

This is the **only** ingest surface with a trail. `/cms/*` is public and un-keyed, so an
upload there has no actor to record.

---

## 3. Route reference

| Method | Route | `examIds` | Notes |
|---|---|---|---|
| `POST` | `/sme/content/pyq` | **required** | Create one PYQ |
| `POST` | `/sme/content/pyq/bulk` | **required per item** | 1–100, all-or-nothing |
| `PATCH` | `/sme/content/pyq/:id` | optional, validated when present | Merge |
| `GET` | `/sme/content/pyq` | — | Paginated list, `?exam=` |
| `POST` | `/sme/content/mains` | **required** | Create one Mains question |
| `POST` | `/sme/content/mains/bulk` | **required per item** | 1–100, all-or-nothing |
| `PATCH` | `/sme/content/mains/:id` | optional, validated when present | Merge |
| `GET` | `/sme/content/mains` | — | Paginated list, `?exam=` |
| `POST` | `/sme/content/simulations` | **required** | Create a mock test (the container) |
| `PATCH` | `/sme/content/simulations/:id` | optional, validated when present | Merge |
| `GET` | `/sme/content/simulations` | — | All simulations incl. inactive, `?exam=` |
| `POST` | `/sme/content/simulations/questions/bulk` | **none — by design** | 1–500, skips duplicate `externalId` |
| `POST` | `/sme/content/simulations/:id/assign-questions` | **none — by design** | Replaces the list, sets `totalQuestions` |

`:id` is parsed by `ParseUUIDPipe` on every route that takes one — a non-UUID is a `400`
before the handler runs.

---

### 3.1 `POST /sme/content/pyq`

Field-for-field the same body as `POST /cms/pyq` (the field tables live in
[CMS_ADMIN_API.md](./CMS_ADMIN_API.md#field-reference)), plus the required `examIds`.

> 🔴 **CORRECTED 2026-09-21 from a live staging call — three more fields are required
> than this doc used to list.** `tags`, `keywords` and `relatedTopics` are **required and
> must be non-empty**. The previous version of this example omitted all three, and that
> exact body is a **400**:
>
> ```json
> {
>   "success": false,
>   "message": [
>     "each value in tags must be a string", "tags must be an array", "tags should not be empty",
>     "each value in keywords must be a string", "keywords must be an array", "keywords should not be empty",
>     "each value in relatedTopics must be a string", "relatedTopics must be an array", "relatedTopics should not be empty"
>   ],
>   "error": "Bad Request",
>   "statusCode": 400,
>   "timestamp": "2026-09-21T14:49:40.168Z",
>   "path": "/api/v1/sme/content/pyq",
>   "method": "POST"
> }
> ```
>
> Note this `message` **is an array** (it comes from `ValidationPipe`), unlike the
> single-string exam errors in §2.3. The curl below has been corrected and re-run
> successfully.

⚠️ **Required, per `prisma/schema.prisma` (`pyq_papers`):** `examIds`, `examName`, `year`,
`subject`, `topic`, `difficulty`, `source`, `nature`, `question`, `correctAnswer`
(**since 2026-10-01: unless `boardDeleted: true`**, see §9.2),
`totalMarks`, `duration`, **`tags`, `keywords`, `relatedTopics`**. The create DTO is
generated from the model, so every one of those
is NOT NULL with no default and a body missing any of them is a `400`. `source` is one of
`NCERT_STANDARD_REFERENCE_BOOK` \| `PYQ_THEME` \| `CURRENT_AFFAIRS` \|
`UNCONVENTIONAL_SOURCE`; `nature` is one of `CORE` \| `CORE_PLUS` \| `NEWS` \| `NEWS_PLUS`
\| `WILD_CARD`. Note that the option fields are *nullable* — `optionA`/`optionB` are not
enforced by the API.

```bash
curl -X POST {{BASE_URL}}/api/v1/sme/content/pyq \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{
    "examIds": ["appsc-group-1"],
    "examName": "SME-HANDOFF-SMOKE",
    "year": 2024,
    "paperNumber": 1,
    "subject": "Polity",
    "topic": "State Legislature",
    "difficulty": "MEDIUM",
    "source": "PYQ_THEME",
    "nature": "CORE",
    "question": "The Andhra Pradesh Legislative Council was revived in which year?",
    "optionA": "2005",
    "optionB": "2007",
    "optionC": "2010",
    "optionD": "2014",
    "correctAnswer": "B",
    "answerExplanation": "The Council was revived on 30 March 2007.",
    "totalMarks": 2,
    "duration": 72,
    "tags": ["smoke"],
    "keywords": ["legislative council"],
    "relatedTopics": ["State Legislature"]
  }'
```

**Response `201`** — the created row, raw:
<!-- captured from staging 2026-09-21, backend f6329e6 -->
The body below is the verbatim response to the exact curl above, run against staging on
2026-09-21. (`examName` is `SME-HANDOFF-SMOKE` because this is a real row that now exists
in the staging bank — id `94534edf-ffe3-4b4a-a531-a7e438abdb10`.)

```json
{
  "id": "94534edf-ffe3-4b4a-a531-a7e438abdb10",
  "examName": "SME-HANDOFF-SMOKE",
  "year": 2024,
  "paperNumber": 1,
  "subject": "Polity",
  "topic": "State Legislature",
  "difficulty": "MEDIUM",
  "source": "PYQ_THEME",
  "nature": "CORE",
  "questionNumber": null,
  "question": "The Andhra Pradesh Legislative Council was revived in which year?",
  "optionA": "2005",
  "optionB": "2007",
  "optionC": "2010",
  "optionD": "2014",
  "optionE": null,
  "correctAnswer": "B",
  "answerExplanation": "The Council was revived on 30 March 2007.",
  "solveTip": null,
  "prediction": null,
  "weightage": null,
  "repeatFrequency": 0,
  "totalMarks": 2,
  "duration": 72,
  "markingScheme": null,
  "tags": ["smoke"],
  "keywords": ["legislative council"],
  "relatedTopics": ["State Legislature"],
  "isActive": true,
  "isVerified": false,
  "language": "ENGLISH",
  "createdAt": "2026-09-21T14:49:51.364Z",
  "updatedAt": "2026-09-21T14:49:51.364Z",
  "examIds": ["appsc-group-1"]
}
```

> **The response is the whole row, not a projection** — 34 keys, `examIds` **last**, and
> every unset nullable column present as `null` rather than omitted. `repeatFrequency`
> defaults to `0` and `language` to `"ENGLISH"`; neither is settable on create.

`examIds` on the response is the **normalised, stored** value — read it back rather than
assuming what you sent is what landed.

---

### 3.2 `POST /sme/content/pyq/bulk`

```bash
curl -X POST {{BASE_URL}}/api/v1/sme/content/pyq/bulk \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "examIds": ["appsc-group-1"], "examName": "APPSC Group 1", "year": 2024,
        "subject": "History", "topic": "Modern India", "difficulty": "EASY",
        "source": "PYQ_THEME", "nature": "CORE",
        "question": "…", "optionA": "…", "optionB": "…", "correctAnswer": "B",
        "totalMarks": 2, "duration": 72 },
      { "examIds": ["*"], "examName": "UPSC CSE", "year": 2024,
        "subject": "Environment", "topic": "Climate", "difficulty": "MEDIUM",
        "source": "CURRENT_AFFAIRS", "nature": "NEWS",
        "question": "…", "optionA": "…", "optionB": "…", "correctAnswer": "A",
        "totalMarks": 2, "duration": 72 }
    ]
  }'
```

**Response `201`:**

```jsonc
// <!-- shape verified against code @bfae389 — bulk create was NOT run on staging, see note -->
{ "count": 2 }
```

> ℹ️ **Not captured.** A single smoke row was created on staging for §3.1 (and is still
> there); a *bulk* run would have written up to 100 more rows into a shared content bank
> for no additional information — the response is one integer. The items themselves take
> the **same corrected required-field set as §3.1**: each item needs `tags`, `keywords`
> and `relatedTopics` too, and the example items above are abbreviated with `"…"`, not
> complete bodies.

A mixed-exam batch like the one above is legal and is exactly what a shared-content dump
looks like. See §4 for the validation order.

---

### 3.3 `PATCH /sme/content/pyq/:id`

Send only what changes. `examIds` is left alone unless you send it.

```bash
# retag a paper that landed in the wrong catalogue
curl -X PATCH {{BASE_URL}}/api/v1/sme/content/pyq/3f6c0e2a-… \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{ "examIds": ["appsc-group-1"] }'

# fix the question text
curl -X PATCH {{BASE_URL}}/api/v1/sme/content/pyq/3f6c0e2a-… \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{ "question": "…corrected text…" }'
```

**Response `200`** — the updated row, same shape as §3.1. **`404`** if the id does not exist.

> 🔴 **CORRECTED 2026-09-21 — you cannot verify a question through this route.** The
> previous version of this doc showed `-d '{ "isVerified": true }'` here. Run against
> staging, that is a **400**:
>
> ```json
> {
>   "success": false,
>   "message": ["property isVerified should not exist"],
>   "error": "Bad Request",
>   "statusCode": 400,
>   "timestamp": "2026-09-21T14:50:13.400Z",
>   "path": "/api/v1/sme/content/pyq/94534edf-ffe3-4b4a-a531-a7e438abdb10",
>   "method": "PATCH"
> }
> ```
>
> `UpdatePYQPaperDto` is generated from the Prisma model's *writable content* fields and
> carries **no `isVerified`, no `isActive`, no `language`, no `repeatFrequency`** —
> while `forbidNonWhitelisted` rejects the whole request on the mere presence of the key.
> The four flags are readable on every response and settable on **none** of them.
>
> **So there is no verification workflow on `/sme/content/*` today.** A portal "Verify"
> button has no endpoint behind it. The updatable set is exactly: `examName`, `year`,
> `paperNumber`, `subject`, `topic`, `difficulty`, `source`, `nature`, `questionNumber`,
> `question`, `optionA`–`optionE`, `correctAnswer`, `answerExplanation`, `solveTip`,
> `prediction`, `weightage`, `totalMarks`, `duration`, `markingScheme`, `tags`,
> `keywords`, `relatedTopics`, `examIds` — **plus, since 2026-10-01, `paperLabel`,
> `questionImageUrl`, `imageCaption`, `boardDeleted`** (§9). A PATCH that touches
> `correctAnswer` or `boardDeleted` is checked against the **merged** row (§9.2).

---

### 3.4 `GET /sme/content/pyq`

Identical filters to `GET /cms/pyq`.

| Param | Default | Notes |
|---|---|---|
| `exam` | — | Exam slug. Matches that exam **or** `*`. Omit for every exam. Unknown slug = **400**. |
| `subject` | — | Case-insensitive `contains` |
| `year` | — | Exact match |
| `difficulty` | — | `EASY` \| `MEDIUM` \| `HARD` \| `EXPERT` |
| `search` | — | Case-insensitive `contains` on the question text |
| `isActive` | — | `true` / `false` |
| `isVerified` | — | `true` / `false` |
| `page` | `1` | min 1 |
| `limit` | `10` | min 1, max 100 |

```bash
curl -H "x-api-key: $API_KEY_SECRET" \
  "{{BASE_URL}}/api/v1/sme/content/pyq?exam=appsc-group-1&isVerified=false&page=1&limit=20"
```

**Response `200`:**

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/content/pyq?exam=appsc-group-1&search=Legislative%20Council&page=1&limit=20` —
verbatim (the one match is the §3.1 smoke row; `data[]` rows are the **full 34-key row**
shown in §3.1, abbreviated here to the keys this section is about):

```jsonc
{
  "data": [
    {
      "id": "94534edf-ffe3-4b4a-a531-a7e438abdb10",
      "examName": "SME-HANDOFF-SMOKE",
      "year": 2024,
      "question": "The Andhra Pradesh Legislative Council was revived in which year?",
      "correctAnswer": "B",
      "tags": ["smoke"], "keywords": ["legislative council"], "relatedTopics": ["State Legislature"],
      "isActive": true,
      "isVerified": false,
      "language": "ENGLISH",
      "createdAt": "2026-09-21T14:49:51.364Z",
      "updatedAt": "2026-09-21T14:49:51.364Z",
      "examIds": ["appsc-group-1"]
      // … the remaining keys of §3.1's row, same order
    }
  ],
  "meta": { "total": 1, "page": 1, "limit": 20, "totalPages": 1, "hasNext": false, "hasPrev": false }
}
```

> **The envelope here is `{ data, meta }` with `meta.total` / `totalPages` / `hasNext` /
> `hasPrev`** — *not* the `{ data, total, page, limit, hasMore }` envelope every
> `/sme/users`, `/sme/orders` and `/sme/transactions` list uses. Two different pagination
> contracts live in this API; do not share one client helper between them.

⚠️ **`?exam=` is validated, not coerced.** An unknown slug is a `400`, never an empty page
that reads as "this exam has no PYQs". Ordering is `year desc, questionNumber asc`.

---

### 3.5 Mains — `POST /sme/content/mains`, `mains/bulk`, `PATCH mains/:id`, `GET mains`

Same four shapes, same rules, same filters. Two differences from PYQ, both inherited from
`CmsMainsService` rather than added here:

- **No options and no `correctAnswer`.** Mains has a free-text `answer` instead.
- **`isOptional` is derived.** If the body omits it, it is set to
  `subject.endsWith(" Optional")`. This is the same rule the production backfill used and
  it runs on this surface too — an optional-subject dump that does not send the flag still
  ends up on the right side of the Mains scope toggle.

> 🔴 **CORRECTED 2026-09-21 from a live staging probe — `marks` is NOT accepted on
> create, and three array fields are required.** `POST /sme/content/mains` with
> `{"marks":15}` returns, verbatim (nothing was written — this is a validation failure):
>
> ```json
> {
>   "success": false,
>   "message": [
>     "property marks should not exist",
>     "examName must be a string", "examName should not be empty",
>     "year must be an integer number", "year should not be empty",
>     "subject must be a string", "subject should not be empty",
>     "topic must be a string", "topic should not be empty",
>     "difficulty must be one of the following values: EASY, MEDIUM, HARD, EXPERT",
>     "difficulty should not be empty",
>     "question must be a string", "question should not be empty",
>     "each value in tags must be a string", "tags must be an array", "tags should not be empty",
>     "each value in keywords must be a string", "keywords must be an array", "keywords should not be empty",
>     "each value in relatedTopics must be a string", "relatedTopics must be an array", "relatedTopics should not be empty"
>   ],
>   "error": "Bad Request",
>   "statusCode": 400,
>   "timestamp": "2026-09-21T14:52:43.130Z",
>   "path": "/api/v1/sme/content/mains",
>   "method": "POST"
> }
> ```
>
> `CreateMainsPaperDto` has **no `marks` property at all**, and `forbidNonWhitelisted`
> rejects on the key's mere presence — so a mains question's `marks` takes the column
> default (`10`) and **cannot be set on create or on PATCH**. The example below has been
> corrected accordingly. `isOptional` *is* settable, and is derived when omitted.

⚠️ **Required, per `prisma/schema.prisma` (`mains_papers`):** `examIds`, `examName`, `year`,
`subject`, `topic`, `difficulty`, `question`, **`tags`, `keywords`, `relatedTopics`**.
`marks` is **not settable** (see above; the column defaults to `10`); `answer` is optional
(a question can be loaded before its model answer exists).

```bash
curl -X POST {{BASE_URL}}/api/v1/sme/content/mains \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{
    "examIds": ["appsc-group-1"],
    "examName": "APPSC Group 1 Mains",
    "year": 2024,
    "subject": "Telugu Literature Optional",
    "topic": "Modern Poetry",
    "difficulty": "HARD",
    "question": "Discuss the influence of the Bhavakavitvam movement on modern Telugu poetry.",
    "answer": "The Bhavakavitvam movement…",
    "tags": ["15M"],
    "keywords": ["Telugu literature"],
    "relatedTopics": ["Modern Poetry"]
  }'
```

**Response `201`** — the created row.
<!-- captured from staging 2026-09-21, backend f6329e6 -->
A mains create was **not** run on staging (only the §3.1 PYQ smoke row was), so the body
below is a real mains row of the identical shape, read back from
`GET /sme/content/mains?exam=upsc-cse&limit=1` on 2026-09-21. Long text is truncated with
`…`; **no key has been removed**:

```json
{
  "id": "755d3cd8-b3ac-489c-a78b-9fb5b8a55607",
  "examName": "UPSC CSE Mains",
  "year": 2025,
  "subject": "GS Paper 1",
  "topic": "History",
  "difficulty": "MEDIUM",
  "questionNumber": null,
  "marks": 10,
  "question": "The sculptors filled the Chandella artform with resilient vigor and breadth of life' Elucidate",
  "answer": "**Question:** “The sculptors filled the Chandella artform wi…",
  "answerExplanation": "### Directive Decoding  \n- Directive check: Note that “Elucidate” asks…",
  "solveTip": "**Tips**  \n\n- Research the **Chandella dynasty** and their patronage…",
  "prediction": null,
  "tags": ["10M"],
  "keywords": ["History"],
  "relatedTopics": ["Art & Culture"],
  "isActive": true,
  "isVerified": false,
  "language": "ENGLISH",
  "isOptional": false,
  "examIds": ["upsc-cse"],
  "createdAt": "2026-03-18T12:02:22.405Z",
  "updatedAt": "2026-03-18T12:02:22.405Z"
}
```

> That same call reported `"meta": { "total": 7466, "page": 1, "limit": 1, "totalPages":
> 7466, "hasNext": true, "hasPrev": false }` — i.e. the staging mains bank holds 7,466
> UPSC rows and **zero APPSC rows** (`?exam=appsc-group-1` → `{"data":[],"meta":{"total":0,…}}`).
> Note `marks: 10` on a row nobody could have set it on.

---

### 3.6 `POST /sme/content/simulations`

The simulation is the **container**. Its `examIds` is what scopes every question assigned
into it (§5).

| Field | Type | Required | Default |
|---|---|---|---|
| `examIds` | string[] | **yes** | — |
| `name` | string | yes | — |
| `durationMinutes` | integer ≥ 1 | yes | — |
| `description` | string | no | — |
| `totalQuestions` | integer ≥ 0 | no | `0`, then overwritten by `assign-questions` |
| `correctMark` | float | no | `2.0` |
| `wrongMark` | float | no | `-0.6666667` |
| `skippedMark` | float | no | `0.0` |
| `color` | string | no | `#A8B5FF` |
| `sortOrder` | integer | no | `0` |
| `isActive` | boolean | no | `true` |

```bash
curl -X POST {{BASE_URL}}/api/v1/sme/content/simulations \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{
    "examIds": ["appsc-group-1"],
    "name": "APPSC Group 1 — Full Mock 1",
    "description": "150 questions, full syllabus",
    "durationMinutes": 150,
    "correctMark": 1.0,
    "wrongMark": -0.33,
    "sortOrder": 1
  }'
```

**Response `201`** — the created simulation.
<!-- captured from staging 2026-09-21, backend f6329e6 -->
A simulation create was **not** run on staging (it would leave a live mock test in the
bank), so the body below is a real simulation row of the identical shape, read back from
`GET /sme/content/simulations` on 2026-09-21 — 14 keys, `examIds` last:

```json
{
  "id": "ab3550c7-35db-4319-944b-5a2cd39aabce",
  "name": "UPSC Prelims 2026 Prediction Test",
  "description": "87 questions we predicted that matched the actual UPSC Prelims 2026 paper (GS Paper-I, 24 May 2026) — 21 direct hits + 66 topic/sub-topic matches, verified against the real paper.",
  "durationMinutes": 105,
  "totalQuestions": 87,
  "correctMark": 2,
  "wrongMark": -0.6666667,
  "skippedMark": 0,
  "color": "#FFD479",
  "sortOrder": 1,
  "isActive": true,
  "createdAt": "2026-06-03T09:05:37.652Z",
  "updatedAt": "2026-06-03T09:05:37.743Z",
  "examIds": ["upsc-cse"]
}
```

The **required-field check** was exercised live — `POST /sme/content/simulations` with
every optional field but neither `name` nor `durationMinutes` (nothing was written):

```json
{
  "success": false,
  "message": [
    "name must be a string",
    "durationMinutes must not be less than 1",
    "durationMinutes must be an integer number"
  ],
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:53:45.266Z",
  "path": "/api/v1/sme/content/simulations",
  "method": "POST"
}
```

> Note the probe sent `totalQuestions`, `correctMark`, `wrongMark`, `skippedMark`,
> `color`, `sortOrder` and `isActive` and **none of them was rejected** — unlike `marks`
> on mains (§3.5), every optional field in the table above really is accepted on create.

`PATCH /sme/content/simulations/:id` takes the same fields, all optional, with §2.4 merge
semantics. **`404`** if the simulation does not exist.

---

### 3.7 `GET /sme/content/simulations`

Returns **every** simulation including inactive ones, ordered `sortOrder asc, createdAt asc`.
Not paginated — it is a plain array.

| Param | Notes |
|---|---|
| `exam` | Exam slug. Matches that exam **or** `*`. Omit for every exam. Unknown slug = **400**. |

```bash
curl -H "x-api-key: $API_KEY_SECRET" \
  "{{BASE_URL}}/api/v1/sme/content/simulations?exam=appsc-group-1"
```

<!-- captured from staging 2026-09-21, backend f6329e6 -->
Both calls, verbatim:

```jsonc
// GET /sme/content/simulations?exam=appsc-group-1   →  200
[]

// GET /sme/content/simulations                     →  200, 6 rows (2 of 6 shown, rest elided)
[
  { "id": "ab3550c7-35db-4319-944b-5a2cd39aabce", "name": "UPSC Prelims 2026 Prediction Test",
    "description": "87 questions we predicted that matched the actual UPSC Prelims 2026 paper…",
    "durationMinutes": 105, "totalQuestions": 87,
    "correctMark": 2, "wrongMark": -0.6666667, "skippedMark": 0,
    "color": "#FFD479", "sortOrder": 1, "isActive": true,
    "createdAt": "2026-06-03T09:05:37.652Z", "updatedAt": "2026-06-03T09:05:37.743Z",
    "examIds": ["upsc-cse"] },
  { "id": "0ce67470-8e9f-4539-b20c-8bb795e0eb2b", "name": "Simulation - 01",
    "description": "Auto-generated mock test #1 from MCQ.xlsx",
    "durationMinutes": 120, "totalQuestions": 100,
    "correctMark": 2, "wrongMark": -0.6666667, "skippedMark": 0,
    "color": "#A8B5FF", "sortOrder": 2, "isActive": true,
    "createdAt": "2026-05-08T10:07:08.264Z", "updatedAt": "2026-06-03T09:05:37.813Z",
    "examIds": ["upsc-cse"] }
  // … 4 more, same shape
]
```

> **`exam=appsc-group-1` returning `[]` is a correct answer, not a filter bug** — all six
> staging simulations are tagged `["upsc-cse"]`, and none carries the `*` sentinel. This
> is also the live proof that an empty result and a bad slug are distinguishable: the
> unknown slug `appsc-grp-1` is a **400** (§2.3), this is a **200 with `[]`**.

---

### 3.8 `POST /sme/content/simulations/questions/bulk`

Creates rows in the shared simulation question bank. **1–500 items** (not 100 — this is the
one cap that differs from the PYQ/Mains bulks). Duplicates are skipped on `externalId`, so
a re-run is safe.

| Field | Required | Notes |
|---|---|---|
| `subject`, `difficulty`, `question`, `optionA`–`optionD`, `correctAnswer` | yes | `correctAnswer` is `A`\|`B`\|`C`\|`D`, upper-cased server-side |
| `externalId` | no | The idempotency key for re-imports |
| `topic`, `microTopic`, `questionType`, `explanation` | no | — |
| `pattern` | no | Normalised server-side: `multi-stmt` → `MULTI_STATEMENT`, `assertion-reason` → `ASSERTION_REASONING`, otherwise upper-cased with `-`/space → `_` |
| `isCurrentAffairs` | no | default `false` |
| `tags` | no | default `[]` |
| `isVerified` | no | default `false` |
| `isActive` | no | default `true` |

```bash
curl -X POST {{BASE_URL}}/api/v1/sme/content/simulations/questions/bulk \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "externalId": "APPSC-M1-001", "subject": "Polity", "difficulty": "MEDIUM",
        "pattern": "multi-stmt",
        "question": "Consider the following statements…",
        "optionA": "1 only", "optionB": "2 only", "optionC": "Both", "optionD": "Neither",
        "correctAnswer": "C", "explanation": "…" }
    ]
  }'
```

```jsonc
// <!-- shape verified against code @bfae389 — bank write NOT run on staging, see note -->
{ "count": 1, "skipped": 0 }
```

The **cap** was exercised live instead — `POST /sme/content/simulations/questions/bulk`
with `{}` (nothing written):
<!-- captured from staging 2026-09-21, backend f6329e6 -->
```json
{
  "success": false,
  "message": [
    "items must contain no more than 500 elements",
    "items must contain at least 1 elements"
  ],
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:53:45.348Z",
  "path": "/api/v1/sme/content/simulations/questions/bulk",
  "method": "POST"
}
```
— confirming the **500** ceiling on the deployed build (the PYQ/Mains bulks cap at 100).

⚠️ **No `examIds` here, and that is not an oversight** — see §5.

---

### 3.9 `POST /sme/content/simulations/:id/assign-questions`

**Replaces** the simulation's whole question list. Order is the array index (1-based
`position`), and `totalQuestions` is set to the array length. Transactional.

```bash
curl -X POST {{BASE_URL}}/api/v1/sme/content/simulations/b0ac5f18-…/assign-questions \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{ "questionIds": ["7a1e…", "8b2f…", "9c30…"] }'
```

```jsonc
// <!-- shape verified against code @bfae389 — a real assign would rewrite a live mock test, not run -->
{ "simulationId": "b0ac5f18-…", "totalQuestions": 3, "assigned": 3 }
```

> ℹ️ **Not captured, deliberately.** This route **replaces** a simulation's whole question
> list, and every simulation on staging is a populated UPSC mock (§3.7) — running it would
> have destroyed one. The **validation** path was exercised instead:
> `POST /sme/content/simulations/ab3550c7-…/assign-questions` with `{}` →
<!-- captured from staging 2026-09-21, backend f6329e6 -->
> ```json
> {
>   "success": false,
>   "message": [
>     "each value in questionIds must be a string",
>     "questionIds must contain at least 1 elements",
>     "questionIds must be an array"
>   ],
>   "error": "Bad Request",
>   "statusCode": 400,
>   "timestamp": "2026-09-21T14:53:45.433Z",
>   "path": "/api/v1/sme/content/simulations/ab3550c7-35db-4319-944b-5a2cd39aabce/assign-questions",
>   "method": "POST"
> }
> ```
>
> **`questionIds` must contain at least 1 element**, so an empty array is *not* how you
> clear a simulation — there is no API for that.

| Status | Cause |
|---|---|
| `400` | `Question IDs not found: <ids>` — one or more ids are not in the bank |
| `400` | `Duplicate question IDs in payload: <ids>` — the same id twice (the link table has a composite PK) |
| `404` | `Simulation <id> not found` |

It is a replace, not an append: sending a shorter list **unassigns** the questions you left
out. They are not deleted — they go back to being unassigned bank rows.

---

## 4. Bulk semantics

Three rules, and the third is the one that matters operationally:

1. **1–100 items** per `pyq/bulk` / `mains/bulk` request. (Simulation *questions* are
   1–500, §3.8.) Over the cap is a `400` from the validation pipe.
2. **Per-item `examIds`.** Each item carries its own; a mixed-exam batch is legal.
3. **All-or-nothing, and the exam check runs FIRST.** Every item's `examIds` is resolved
   before `bulkCreate` is called at all, so a batch with one bad slug in item 73 writes
   **nothing** — you never have to work out which half of a 100-row dump landed. Past that
   gate, the insert itself is a single Prisma transaction, so a database-level failure is
   also all-or-nothing.

The audit row for a bulk records the **union** of the exams touched:

```jsonc
{ "route": "POST /sme/content/mains/bulk", "count": 100, "examIds": ["appsc-group-1", "*"] }
```

---

## 5. The simulations model: the container carries the exam

```
simulations            → HAS examIds        (the mock test)
  └─ SimulationQuestionLink (position)
       └─ simulation_questions  → NO examIds  (the shared bank)
```

`simulation_questions` has **no exam column**, by design. A question reaches an exam only by
being assigned into a simulation, and the simulation carries the tag. Three consequences:

- Questions created by `questions/bulk` are **inert** until assigned — published nowhere,
  belonging to no exam.
- A question assigned into two simulations belonging to two exams appears in both. The bank
  is genuinely shared.
- The exam filter on `GET /sme/content/question-quality` reaches simulation questions
  *through their links* — so a question in no simulation is unreachable under `?exam=`.
  That is correct, not a gap (see [SME_INSIGHTS_API.md](./SME_INSIGHTS_API.md) §1).

The build order is therefore always: **create the simulation → bulk-create the questions →
assign them.**

---

## 6. How to verify a load

Do not trust the `201`. After any bulk load:

```bash
curl -H "x-api-key: $API_KEY_SECRET" \
  "{{BASE_URL}}/api/v1/sme/exams/appsc-group-1/content-counts"
```

Read the **`exclusive`** column — content authored *for* this exam, as opposed to `shared`
(`*`-tagged) content it merely inherits. `exclusive: 0` on a module you just loaded into is
the signature of a mis-tag. (On `/sme/content/*` an *untagged* load is impossible, so if
this happens the slug was wrong-but-real: you tagged the rows for a different exam.)

Cross-check the rows themselves:

```bash
curl -H "x-api-key: $API_KEY_SECRET" \
  "{{BASE_URL}}/api/v1/sme/content/pyq?exam=appsc-group-1&limit=5"
```

Full field semantics for `content-counts` are in
[SME_EXAMS_API.md](./SME_EXAMS_API.md) §2; the launch checklist that wraps this step is §6
of the same doc.

---

## 7. What this surface does NOT mirror

Deliberately — these have no exam dimension to get wrong, so there was nothing to gate:

| Not here | Keep using |
|---|---|
| Delete a question or a simulation | `DELETE /cms/pyq/:id`, `DELETE /cms/mains/:id`, `DELETE /cms/simulations/:id` |
| Fetch one question by id | `GET /cms/pyq/:id`, `GET /cms/mains/:id` |
| Resolve `externalId` → UUID after a bulk insert | `GET /cms/simulations/questions?externalIds=…` |
| Psychometric questions and test sets | `/cms/psychometric/*` — psychometric content carries **no `examIds`** at all; an exam opts out of the test with the `psychometric` feature flag instead ([SME_EXAMS_API.md](./SME_EXAMS_API.md) §7) |
| Library study documents | `/content-doc-admin` ([CONTENT_DOC_SME_API.md](./CONTENT_DOC_SME_API.md)) — already exam-aware, `examIds` optional |
| Reels | `PUT /reels/bulk` — `examIds` per video |

---

## 8. Common errors

| Status | Message | Fix |
|---|---|---|
| `400` | `examIds is required: list the exam slugs this content belongs to, or ["*"] for every exam.` | Add `examIds` to the body (or to the offending bulk item) |
| `400` | `Unknown exam "<slug>". Create it via POST /sme/exams first.` | Check the slug against `GET /sme/exams`; create it if it is genuinely new |
| `400` | `examIds must be an array of exam slugs, e.g. ["appsc-group-1"] or ["*"].` | You sent a bare string — it must be an array |
| `400` | `Question IDs not found: …` / `Duplicate question IDs in payload: …` | `assign-questions` payload problem (§3.9) |
| `400` | array of validator strings | An ordinary field-level validation failure from the global pipe — note the shape difference from the two `examIds` 400s above |
| `400` | `correctAnswer is required unless boardDeleted is true — only a question the board deleted may have no answer.` | A live PYQ needs one letter (§9.2). On bulk the message is prefixed `items[<i>]: ` |
| `400` | `correctAnswer must be a single option letter A–E, or null for a question the board deleted (got "3").` | Send the **letter**, uppercase, not the option number (§9.2) |
| `401` | — | Missing or wrong `x-api-key` |
| `404` | `<Resource> not found` | The `:id` does not exist |

---

## 9. PYQ figures, paper labels and board-deleted questions (2026-10-01)

Four optional columns on `pyq_papers`, accepted on `POST /sme/content/pyq`, `pyq/bulk` and
`PATCH pyq/:id` (and on `/cms/pyq/*`). All four are **additive**: an omitted field means
"as before", and every existing UPSC row reads `null` / `false`. What the app receives
is specified, with real serialized responses, in
[PYQ_MOBILE_CONTRACT_2026-10.md](./PYQ_MOBILE_CONTRACT_2026-10.md).

### 9.1 The fields

| Field | Type | Default | Meaning |
|---|---|---|---|
| `paperLabel` | string \| null | `null` | The paper chip shown beside the year. **Shown verbatim**, and the server does not validate it, so keep to the convention: `PAPER-I`, `PAPER-II`, `SET-A`, `SET-B`, `MARCH`, `OCTOBER`, `SEPTEMBER` (APPSC uses all of them: a single-paper year is `PAPER-I`). `null` = no chip (UPSC). |
| `questionImageUrl` | string \| null | `null` | Public URL of the figure in the question stem (map, diagram, table-as-image). The app shows it under the stem, with a tap-to-zoom view on a white background. Use a PNG with a transparent or white background and dark line art. Upload it via §9.4; a URL that does not load shows a broken image. |
| `imageCaption` | string \| null | `null` | Caption under the figure. Optional even when there is a figure (APPSC figures carry none). |
| `boardDeleted` | boolean | `false` | The board **withdrew** the question after the exam. It stays in the bank and is **shown** (list, year folder, details) but is **inert**. See §9.3. |

```bash
# a figure question
curl -X PATCH {{BASE_URL}}/api/v1/sme/content/pyq/<id> \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{ "paperLabel": "PAPER-II", "questionImageUrl": "https://<bucket>.s3.ap-south-1.amazonaws.com/pyq-figures/<uuid>-appsc-2024-p2-q3.png" }'

# the board withdrew a live question
curl -X PATCH {{BASE_URL}}/api/v1/sme/content/pyq/<id> \
  -H "x-api-key: $API_KEY_SECRET" -H "Content-Type: application/json" \
  -d '{ "boardDeleted": true, "correctAnswer": null }'
```

### 9.2 The answer-key write rule

> **A live question needs exactly one option letter `A`–`E`. Only a question with
> `boardDeleted: true` may have `correctAnswer: null`.**

- Enforced on every PYQ write: `POST`/`PATCH` on `/sme/content/pyq`, `pyq/bulk`, `/cms/pyq`,
  `/cms/pyq/bulk` and `/pyq`.
- The letter must be **uppercase `A`–`E`**. `"3"`, `"c"` and `"(C)"` are all `400`. The app
  ticks the option whose label equals `correctAnswer`, so a number marks every answer wrong.
  That shipped unnoticed for two months in 2026, which is why the server now refuses it.
- **PATCH is checked against the merged row.** `{ "correctAnswer": null }` on a live row is a
  `400`, and so is `{ "boardDeleted": false }` on a row that has no answer. To withdraw a
  question send both `boardDeleted: true` and `correctAnswer: null` (sending only
  `boardDeleted: true` is also accepted; the stored letter is then never served).
- The letter **format** is only checked on a value you actually send, so a typo fix on an
  old row never fails over a legacy value you did not touch.
- **Bulk is all-or-nothing** and the message names the item: `items[3]: correctAnswer is
  required unless boardDeleted is true — …`.

Verbatim (single-string `message`, unlike the validator arrays in §3.1):

```json
{ "success": false,
  "message": "correctAnswer is required unless boardDeleted is true — only a question the board deleted may have no answer.",
  "error": "Bad Request", "statusCode": 400 }
```
```json
{ "success": false,
  "message": "correctAnswer must be a single option letter A–E, or null for a question the board deleted (got \"3\").",
  "error": "Bad Request", "statusCode": 400 }
```

### 9.3 What a board-deleted question does in the app, and `409 QUESTION_WITHDRAWN`

Withdrawn means `boardDeleted = true` **or** `correctAnswer` is `NULL`; both are treated the same.

- **Shown:** list rows, the year folder and question details still include it. `correctAnswer`,
  the explanation and the solve tip are `null` for **every** tier, premium included.
- **Never** in a mock test, and **excluded** from `/pyq/metrics` totals, so 100 % stays reachable.
- **Refused** with `409` (no credit is spent on reveal): submitting an answer, revealing the
  explanation, and **adding** a bookmark. Removing a bookmark saved before the withdrawal still
  works.

```json
{
  "success": false,
  "message": "This question was withdrawn by the board and cannot be attempted.",
  "error": "Conflict",
  "code": "QUESTION_WITHDRAWN",
  "statusCode": 409,
  "timestamp": "<ISO-8601>",
  "path": "<the request path>",
  "method": "POST"
}
```

Key on `code`, not `message`. These 409s come from the **app** routes (`/pyq/:id/submit`,
`/pyq/:id/reveal`, `/pyq/:id/bookmark`). Nothing on `/sme/*` returns them.

**Portal-side counts:**
- `GET /sme/exams/:id/content-counts` → the `prelims` row gains **`boardDeleted`**: how many
  of its `visible` rows are withdrawn. They stay inside `visible` / `exclusive`, because
  they are shown.
- `GET /sme/filter-config/prelims?exam=` → `questionCount` counts **playable** rows only,
  while the subject list still comes from every active row.

### 9.4 Uploading a figure — `POST /sme/media/upload-url` with `folder: "pyq-figures"`

The same two-step presigned upload as campaign artwork
([SME_OFFERS_API.md](./SME_OFFERS_API.md) §7):

```json
// 1. POST /sme/media/upload-url   (x-api-key)
{ "filename": "appsc-2024-p2-q3.png", "contentType": "image/png", "folder": "pyq-figures" }
```
```json
// 200
{
  "uploadUrl": "https://<bucket>.s3.ap-south-1.amazonaws.com/pyq-figures/<uuid>-appsc-2024-p2-q3.png?X-Amz-…",
  "publicUrl": "https://<bucket>.s3.ap-south-1.amazonaws.com/pyq-figures/<uuid>-appsc-2024-p2-q3.png",
  "key": "pyq-figures/<uuid>-appsc-2024-p2-q3.png",
  "expiresInSeconds": 300,
  "maxBytes": 5242880
}
```

2. `PUT <uploadUrl>` with header `Content-Type: image/png` (exactly the type you asked
   for, because the signature is bound to it) and the raw bytes as the body. No `x-api-key`
   on this PUT, and it expires after `expiresInSeconds`.
3. Check the image loads: `HEAD <publicUrl>` should return `200` with the same
   `Content-Length`.
4. Store `publicUrl` as the question's `questionImageUrl` (create or PATCH).

- `contentType`: `image/png` \| `image/jpeg` \| `image/webp` only. `folder` must be exactly
  `pyq-figures` (`pyq_figures`, `pyq` and anything else are `400`).
- Every call mints a **new** key, so a re-upload gives a new URL and nothing is overwritten.
  Repoint the question at the new URL.

> 🔴 **`maxBytes` is advertised, not enforced.** The presigned PUT carries no size
> condition, so S3 accepts a file of any size. **Check the size yourself before uploading**
> (keep figures well under 5 MB; the 38 APPSC figures are 1–70 KB each). A 20 MB scan uploaded by
> mistake would be downloaded by every phone that opens the question.

**Errors:** `400` unsupported `contentType` or unknown `folder` · `503` object storage not
configured on that environment. Upload **per environment**: a staging URL must never be
stored on a production row.
