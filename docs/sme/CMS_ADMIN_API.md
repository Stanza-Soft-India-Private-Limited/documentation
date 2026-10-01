# CMS Admin API — Unified Content Management Guide

> LEGACY INGEST SURFACE — kept working, not extended. New integrations should use /sme/content/* (key-gated, exam-safe). Migration map in §0.

> Changed 2026-09-21 — exam-safe ingest: /sme/content/* + examIds on /cms and content-doc (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).
> Changed 2026-09-21 — per-exam broadcast, validated segment exam (+ fix), survey targetExams, feedback ?exam= (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

## Overview

This API provides a unified CMS for managing PYQ (Previous Year Questions), Mains questions, Simulations (mock tests) and Psychometric content. All endpoints live under the `/cms` prefix and are public admin routes — no authentication required. Operations include full CRUD, bulk creation (up to 100 items in an all-or-nothing transaction), and paginated listing with filters.

BASE_URL: https://app.stanzasoft.ai

**Base URL:** `{{BASE_URL}}/api/v1`
**Authentication:** None required (all endpoints are public/admin)
**Note:** All endpoints use the `/api/v1/` global prefix.

**The platform is multi-exam.** Every content row carries an `examIds` array (UPSC / APPSC /
TPSC / …). On `/cms/*` that field is **optional and defaults to `["upsc-cse"]`** — see §0 for
why, and for the key-gated surface that makes forgetting it impossible.

---

## 0. Migrating to `/sme/content/*`

`/cms/*` is public, un-keyed and un-audited, and its `examIds` is optional. That last part
has one specific failure mode: **an APPSC upload that forgets `examIds` succeeds, returns
`201`, and lands every row in the UPSC catalogue.** Nothing errors.

`/sme/content/*` ([SME_CONTENT_INGEST_API.md](./SME_CONTENT_INGEST_API.md)) is the same
three services behind `x-api-key`, with `examIds` **required** on every create and every
write audited as `CONTENT_INGEST`. No logic is duplicated — both surfaces call the same
`CmsPyqService` / `CmsMainsService` / `CmsSimulationService` writes, so they cannot drift.

**`/cms/*` is not deprecated and is not going away** — the existing import scripts depend on
it, and making `examIds` required there would break all of them at once. It is simply not
where new integrations should start.

| `/cms/*` | `/sme/content/*` | difference |
|---|---|---|
| POST /cms/pyq | POST /sme/content/pyq | examIds required + validated; audited |
| POST /cms/pyq/bulk | POST /sme/content/pyq/bulk | as above, per item; whole batch refused before any write |
| PATCH /cms/pyq/:id | PATCH /sme/content/pyq/:id | present examIds must be non-empty + known |
| GET /cms/pyq | GET /sme/content/pyq | identical filters incl. ?exam= |
| (same four for mains) | | |
| POST /cms/simulations | POST /sme/content/simulations | examIds required |
| PATCH /cms/simulations/:id | PATCH /sme/content/simulations/:id | present examIds validated |
| GET /cms/simulations | GET /sme/content/simulations | identical |
| POST /cms/simulations/questions/bulk | POST /sme/content/simulations/questions/bulk | audit only (no examIds: questions inherit the parent's) |
| POST /cms/simulations/:id/assign-questions | same suffix under /sme/content | audit only |
| DELETE /cms/*/:id, GET /cms/{pyq,mains}/:id, GET /cms/simulations/questions | not mirrored | keep using /cms |

Psychometric content is not mirrored either, because it has no exam dimension to get wrong
— see [§ Psychometric and the exam dimension](#psychometric-and-the-exam-dimension).

### `examIds` on `/cms/*` — exactly what it does

| You send | `/cms/*` stores | Notes |
|---|---|---|
| `["appsc-group-1"]` | `["appsc-group-1"]` | |
| `[" APPSC-Group-1 "]` | `["appsc-group-1"]` | **Fixed 2026-09-21.** Previously stored the literal padded string, which matched no filter — the content was invisible in its own catalogue, with no error anywhere. |
| `["upsc-cse", "UPSC-CSE"]` | `["upsc-cse"]` | trimmed, lower-cased, de-duplicated |
| `["*"]` | `["*"]` | every exam, including ones that do not exist yet |
| `["appsc-grp-1"]` — well-formed, but **not a real exam** | `["appsc-grp-1"]` + a server WARN | ⚠️ still the worst case: the value is **stored verbatim** and matches nothing anywhere, and the call still returns `201`. Since 2026-09-21 the catalogue is at least *consulted* and a WARN is logged (see below). `/sme/content/*` 400s on it |
| `["APPSC Group 1"]` — not a valid slug shape | `["upsc-cse"]` + a server WARN | spaces/underscores/punctuation fail the `^[a-z0-9]+(-[a-z0-9]+)*$` check and the value is dropped |
| omitted | `["upsc-cse"]` + a server WARN | this is the trap |
| `[]` on a `PATCH` | `["upsc-cse"]` | an empty list is treated as "unsaid", not as an error (`/sme/content/*` 400s on it) |

**The untagged-ingest warning.** When a create resolves to `["upsc-cse"]` because the field
was absent, or because every slug sent failed the *shape* check, the server logs a WARN
naming the resource — e.g. *"PYQ created with NO examIds — defaulting to [upsc-cse]. If this
is a bulk ingest for another exam, it is landing in the wrong catalogue."* It is a log line,
not a response field: the call still returns `201`. Treat it as a tripwire during a load,
not as something the client can read.

**The unknown-slug warning (added 2026-09-21).** A *second*, different failure — a slug
that passes the shape check but names no exam, e.g. `appsc-grp-1` — is now caught too.
Every resolved slug is looked up in the exam catalogue and an unknown one logs:

> *"PYQ tagged with exam "appsc-grp-1", which is not in the catalogue — the content is
> being stored with that tag anyway and will be invisible to every exam. Check the slug,
> or create the exam via POST /sme/exams."*

It fires on `/cms/pyq`, `/cms/mains` and `/cms/simulations` (create, bulk-create and the
`PATCH`es that send `examIds`) and on the content-doc admin create/update. The `*`
sentinel is skipped — it is never a catalogue entry.

⚠️ **It WARNs, it never 400s, and the row is still stored with the bad tag.** That is
deliberate: every import script in existence calls `/cms/*`, and rejecting would break all
of them at once for a class of error they have been getting away with. So a wrong-but-
well-formed slug remains **invisible in the response** — `201`, no error field, nothing the
client can read — and `content-counts` is still the check that catches it after the fact.
It remains the single strongest reason to load new content through `/sme/content/*`
instead, where the same check is a **400 before anything is written**.

**On `PATCH`, an omitted `examIds` leaves the column alone.** Normalisation only runs when
the key is actually present in the body, so fixing a typo never silently re-tags the row.

### After any bulk load, check `content-counts`

```bash
curl -H "x-api-key: $API_KEY_SECRET" \
  "{{BASE_URL}}/api/v1/sme/exams/appsc-group-1/content-counts"
```

Read the **`exclusive`** column — rows authored *for* that exam, as opposed to `shared`
(`*`-tagged) rows it merely inherits. **`exclusive: 0` on a module you just loaded into is
the signature of a forgotten `examIds`**: those rows are now UPSC content. Fix by
`PATCH`-ing them with the right tags. Field-by-field semantics are in
[SME_EXAMS_API.md](./SME_EXAMS_API.md) §2.

---

## Quick Start

### Create a PYQ Question

```
POST /cms/pyq
Content-Type: application/json
```

**Request Body:**

```json
{
  "examIds": ["upsc-cse"],
  "examName": "UPSC CSE",
  "year": 2024,
  "paperNumber": 1,
  "subject": "Polity",
  "topic": "Fundamental Rights",
  "question": "Which of the following Fundamental Rights is available only to citizens?",
  "optionA": "Right to Equality (Article 14)",
  "optionB": "Right to Freedom of Speech (Article 19)",
  "optionC": "Right to Life (Article 21)",
  "optionD": "Right to Constitutional Remedies (Article 32)",
  "correctAnswer": "B",
  "answerExplanation": "Article 19 rights are available only to citizens, not to foreigners.",
  "solveTip": "Remember: Articles 15, 16, 19, 29, 30 are citizen-only rights.",
  "difficulty": "MEDIUM",
  "questionNumber": 12,
  "totalMarks": 2,
  "tags": ["fundamental-rights", "article-19"],
  "keywords": ["citizen rights", "article 19"],
  "isActive": true,
  "isVerified": false
}
```

**Response (201):** Returns the created question with a generated UUID `id`, `createdAt`, `updatedAt` and the normalised `examIds` actually stored.

**`examIds`** says which exam modes the row belongs to. Omit it and it becomes
`["upsc-cse"]` — which is why an APPSC load that forgets it disappears into the UPSC
catalogue (§0). Use `["*"]` for content shared by **every** exam, including ones that do not
exist yet: `*` is a sentinel, matched by every exam's reads, so shared content never needs
re-tagging when a new exam launches.

---

### Create a Mains Question

```
POST /cms/mains
Content-Type: application/json
```

**Request Body:**

```json
{
  "examIds": ["upsc-cse"],
  "examName": "UPSC CSE Mains",
  "year": 2024,
  "subject": "GS Paper 2",
  "topic": "Indian Polity",
  "question": "Discuss the significance of the Basic Structure Doctrine in protecting the Indian Constitution from arbitrary amendments.",
  "answer": "The Basic Structure Doctrine, established in Kesavananda Bharati v. State of Kerala (1973)...",
  "answerExplanation": "Focus on: supremacy of the Constitution, separation of powers, judicial review...",
  "solveTip": "Structure your answer with: origin, key cases, features covered, significance.",
  "difficulty": "HARD",
  "questionNumber": 5,
  "prediction": "HIGH",
  "tags": ["basic-structure", "constitutional-amendment"],
  "keywords": ["Kesavananda Bharati", "basic structure"],
  "relatedTopics": ["Judicial Review", "Constitutional Amendments"],
  "isActive": true,
  "isVerified": false
}
```

**Response (201):** Returns the created Mains question with a generated UUID `id`.

**Key difference from PYQ:** No options (A/B/C/D/E), no `correctAnswer`, no `source`, no `nature`, no `totalMarks`, no `duration`, no `markingScheme`. Mains has a free-text `answer` field instead.

---

### Create a Psychometric Question

```
POST /cms/psychometric/questions
Content-Type: application/json
```

**Request Body:**

```json
{
  "subject": "Aptitude",
  "topic": "Logical Reasoning",
  "psychometricType": "COGNITIVE",
  "questionNumber": 1,
  "question": "If all roses are flowers and some flowers fade quickly, which conclusion follows?",
  "optionA": "All roses fade quickly",
  "optionB": "Some roses may fade quickly",
  "optionC": "No roses fade quickly",
  "optionD": "All flowers are roses",
  "correctAnswer": "B",
  "difficulty": "MEDIUM",
  "tags": ["syllogism", "logical-reasoning"],
  "keywords": ["deductive reasoning"],
  "isActive": true,
  "isVerified": false
}
```

**Response (201):** Returns the created Psychometric question with a generated UUID `id`.

---

### Create a Psychometric Test Set

```
POST /cms/psychometric/test-sets
Content-Type: application/json
```

**Request Body:**

```json
{
  "name": "Logical Reasoning — Set A",
  "description": "Beginner-level logical reasoning assessment (20 questions)",
  "questionIds": [
    "uuid-of-question-1",
    "uuid-of-question-2",
    "uuid-of-question-3"
  ],
  "isActive": true
}
```

**Required fields:** `name`, `questionIds`
**Optional fields:** `description`, `isActive` (default: true)

**Response (201):** Returns the created test set with its UUID `id`.

---

## Bulk Create (PYQ, Mains, Psychometric Questions)

All three question resource types support bulk creation at their respective `/bulk` endpoint. The format is identical across all resources.

```
POST /cms/pyq/bulk
POST /cms/mains/bulk
POST /cms/psychometric/questions/bulk
Content-Type: application/json
```

**Request Body:**

```json
{
  "items": [
    { "...fields for item 1..." },
    { "...fields for item 2..." }
  ]
}
```

**Rules:**

- Maximum **100 items** per request
- All-or-nothing transaction — if any item fails validation, **none** are created
- Each item in the array follows the same schema as the corresponding single-create endpoint
- **`examIds` is per item** (PYQ and Mains), so a mixed-exam batch is legal — that is what a
  shared-content dump looks like. An item that omits it gets `["upsc-cse"]`, per item, with
  no effect on its neighbours. Psychometric items have no `examIds`.

> Simulation *questions* bulk-create at `POST /cms/simulations/questions/bulk` with a
> different cap (**1–500**) and different semantics (skip-duplicates, not all-or-nothing) —
> see the Simulations section below.

**Response (201):**

```json
{
  "count": 2
}
```

---

## Pagination

All list endpoints return paginated responses in this format:

```json
{
  "data": [
    { "id": "uuid-1", "question": "...", "...": "..." },
    { "id": "uuid-2", "question": "...", "...": "..." }
  ],
  "meta": {
    "total": 150,
    "page": 1,
    "limit": 10,
    "totalPages": 15,
    "hasNext": true,
    "hasPrev": false
  }
}
```

---

## All Endpoints

### PYQ Questions — `/cms/pyq`

| Method   | Endpoint        | Description                                       |
| -------- | --------------- | ------------------------------------------------- |
| `POST`   | `/cms/pyq`      | Create single PYQ question                        |
| `POST`   | `/cms/pyq/bulk` | Bulk create (max 100, all-or-nothing transaction) |
| `GET`    | `/cms/pyq`      | List with filters + pagination                    |
| `GET`    | `/cms/pyq/:id`  | Get single by UUID                                |
| `PATCH`  | `/cms/pyq/:id`  | Update (partial)                                  |
| `DELETE` | `/cms/pyq/:id`  | Hard delete                                       |

---

#### POST /cms/pyq

Create a new PYQ question. Body as shown in Quick Start above.

#### POST /cms/pyq/bulk

Bulk create up to 100 PYQ questions. Body: `{ "items": [...] }`.

#### GET /cms/pyq

List PYQ questions with pagination and filters.

**Query params:**

| Param        | Type    | Default | Description                                    |
| ------------ | ------- | ------- | ---------------------------------------------- |
| `page`       | integer | 1       | Page number (min: 1)                           |
| `limit`      | integer | 10      | Items per page (min: 1, max: 100)              |
| `exam`       | string  | —       | Exam mode slug. Matches that exam **or** `*`. Omit for every exam. Unknown slug = **400** |
| `subject`    | string  | —       | Filter by subject (case-insensitive)           |
| `year`       | integer | —       | Filter by exact exam year                      |
| `difficulty` | enum    | —       | `EASY`, `MEDIUM`, `HARD`, `EXPERT`             |
| `search`     | string  | —       | Search within question text (case-insensitive) |
| `isActive`   | boolean | —       | Filter by active status (`true`/`false`)       |
| `isVerified` | boolean | —       | Filter by verified status (`true`/`false`)     |

Every row in the response carries its `examIds`, so a mis-tagged upload is visible here.

**Request Examples:**

```
GET /cms/pyq?page=1&limit=20
GET /cms/pyq?exam=appsc-group-1&isVerified=false
GET /cms/pyq?subject=History&year=2024
GET /cms/pyq?search=constitution&difficulty=MEDIUM
GET /cms/pyq?isActive=true&isVerified=false&page=2&limit=10
```

#### GET /cms/pyq/:id

Get full PYQ question by UUID. Returns the complete question object.

**Errors:** `404` — Resource not found

#### PATCH /cms/pyq/:id

Update a PYQ question. Send only the fields you want to change.

```json
{
  "difficulty": "HARD",
  "isVerified": true
}
```

**`examIds` on a PATCH:** omit the key and the row's tags are left completely alone —
normalisation only runs when the key is present, so an unrelated edit can never re-tag the
paper. Send it to re-tag:

```json
{
  "examIds": ["appsc-group-1"]
}
```

⚠️ `{"examIds": []}` here resolves to `["upsc-cse"]` rather than erroring.
`PATCH /sme/content/pyq/:id` 400s on the same body.

**Errors:** `404` — Resource not found, `400` — Validation failed

#### DELETE /cms/pyq/:id

Hard delete — permanently removes the question from the database.

**Errors:** `404` — Resource not found

---

### Mains Questions — `/cms/mains`

| Method   | Endpoint          | Description                                       |
| -------- | ----------------- | ------------------------------------------------- |
| `POST`   | `/cms/mains`      | Create single Mains question                      |
| `POST`   | `/cms/mains/bulk` | Bulk create (max 100, all-or-nothing transaction) |
| `GET`    | `/cms/mains`      | List with filters + pagination                    |
| `GET`    | `/cms/mains/:id`  | Get single by UUID                                |
| `PATCH`  | `/cms/mains/:id`  | Update (partial)                                  |
| `DELETE` | `/cms/mains/:id`  | Hard delete                                       |

---

#### POST /cms/mains

Create a new Mains question. Body as shown in Quick Start above.

#### POST /cms/mains/bulk

Bulk create up to 100 Mains questions. Body: `{ "items": [...] }`.

#### GET /cms/mains

List Mains questions with pagination and filters.

**Query params:**

| Param        | Type    | Default | Description                                    |
| ------------ | ------- | ------- | ---------------------------------------------- |
| `page`       | integer | 1       | Page number (min: 1)                           |
| `limit`      | integer | 10      | Items per page (min: 1, max: 100)              |
| `exam`       | string  | —       | Exam mode slug. Matches that exam **or** `*`. Omit for every exam. Unknown slug = **400** |
| `subject`    | string  | —       | Filter by subject (case-insensitive)           |
| `year`       | integer | —       | Filter by exact exam year                      |
| `difficulty` | enum    | —       | `EASY`, `MEDIUM`, `HARD`, `EXPERT`             |
| `search`     | string  | —       | Search within question text (case-insensitive) |
| `isActive`   | boolean | —       | Filter by active status (`true`/`false`)       |
| `isVerified` | boolean | —       | Filter by verified status (`true`/`false`)     |

Every row in the response carries its `examIds`.

**Request Examples:**

```
GET /cms/mains?page=1&limit=20
GET /cms/mains?exam=appsc-group-1&page=1&limit=20
GET /cms/mains?subject=GS Paper 2&year=2024
GET /cms/mains?search=federalism&isVerified=true
```

#### GET /cms/mains/:id

Get full Mains question by UUID. Returns the complete question object.

**Errors:** `404` — Resource not found

#### PATCH /cms/mains/:id

Update a Mains question. Send only the fields you want to change.

```json
{
  "answer": "Updated model answer...",
  "isVerified": true
}
```

Same `examIds` merge rule as PYQ: omitted = untouched, `[]` = `["upsc-cse"]`.

⚠️ **`isOptional` is derived on create, not on update.** `POST /cms/mains` (and the bulk)
sets `isOptional` from a `" Optional"` suffix on `subject` when the body omits it. A `PATCH`
that renames the subject does **not** re-derive it — send `isOptional` explicitly if you
change the subject across that boundary.

**Errors:** `404` — Resource not found, `400` — Validation failed

#### DELETE /cms/mains/:id

Hard delete — permanently removes the question from the database.

**Errors:** `404` — Resource not found

---

### Simulations (mock tests) — `/cms/simulations`

| Method   | Endpoint                                 | Description                                                  |
| -------- | ---------------------------------------- | ------------------------------------------------------------ |
| `GET`    | `/cms/simulations`                       | List ALL simulations including inactive (`?exam=` optional)   |
| `POST`   | `/cms/simulations`                       | Create a simulation                                           |
| `PATCH`  | `/cms/simulations/:id`                   | Update (partial, merge)                                       |
| `DELETE` | `/cms/simulations/:id`                   | **Soft** delete — sets `isActive: false`                      |
| `GET`    | `/cms/simulations/questions`             | Resolve `externalId` → UUID after a bulk insert               |
| `POST`   | `/cms/simulations/questions/bulk`        | Bulk create questions (1–500, skips duplicate `externalId`)   |
| `POST`   | `/cms/simulations/:id/assign-questions`  | Replace the assigned question list; sets `totalQuestions`     |

#### The data model: the container carries the exam

```
simulations            → HAS examIds        (the mock test)
  └─ SimulationQuestionLink (position)
       └─ simulation_questions  → NO examIds  (the shared question bank)
```

`simulation_questions` has **no exam column**. A question reaches an exam only by being
assigned into a simulation, and the simulation carries the tag — so a question created by
`questions/bulk` and never assigned is inert, published nowhere and belonging to no exam.
The build order is always **create the simulation → bulk-create the questions → assign
them**.

#### GET /cms/simulations

Returns every simulation, inactive ones included, ordered `sortOrder asc, createdAt asc`.
**Not paginated** — a plain array. Every row carries its `examIds`.

| Param  | Type   | Description |
| ------ | ------ | ----------- |
| `exam` | string | Exam mode slug. Matches that exam **or** `*`. Omit for every exam. Unknown slug = **400** |

```
GET /cms/simulations
GET /cms/simulations?exam=appsc-group-1
```

#### POST /cms/simulations

| Field             | Type     | Required | Default      | Notes                                                        |
| ----------------- | -------- | -------- | ------------ | ------------------------------------------------------------ |
| `examIds`         | string[] | no       | `["upsc-cse"]` | Exam modes this mock test belongs to. `["*"]` = every exam. **Required** on `POST /sme/content/simulations` |
| `name`            | string   | yes      | —            | Display name                                                  |
| `durationMinutes` | integer  | yes      | —            | Minimum 1                                                     |
| `description`     | string   | no       | —            |                                                               |
| `totalQuestions`  | integer  | no       | `0`          | Overwritten by `assign-questions`                             |
| `correctMark`     | float    | no       | `2.0`        |                                                               |
| `wrongMark`       | float    | no       | `-0.6666667` |                                                               |
| `skippedMark`     | float    | no       | `0.0`        |                                                               |
| `color`           | string   | no       | `#A8B5FF`    | Card colour in the app                                        |
| `sortOrder`       | integer  | no       | `0`          |                                                               |
| `isActive`        | boolean  | no       | `true`       |                                                               |

```json
{
  "examIds": ["appsc-group-1"],
  "name": "APPSC Group 1 — Full Mock 1",
  "description": "150 questions, full syllabus",
  "durationMinutes": 150,
  "correctMark": 1.0,
  "wrongMark": -0.33,
  "sortOrder": 1
}
```

**Response (201):** the created simulation, including the normalised `examIds`.

#### PATCH /cms/simulations/:id

All of the create fields, all optional. `examIds` follows the same merge rule as PYQ/Mains:
omitted = untouched, present = normalised, `[]` = `["upsc-cse"]`.

**Errors:** `404` — `Simulation <id> not found`

#### DELETE /cms/simulations/:id

**Soft** delete — sets `isActive: false` and returns
`{ "message": "Simulation deactivated successfully" }`. Unlike PYQ and Mains, nothing is
destroyed. (`GET /cms/simulations` still lists it.)

#### GET /cms/simulations/questions

Resolves `externalId`s back to database UUIDs — what an import script calls between the
bulk insert and `assign-questions`.

| Param         | Type   | Description                                    |
| ------------- | ------ | ---------------------------------------------- |
| `externalIds` | string | Comma-separated list, e.g. `ECON-001,POL-002`  |

```
GET /cms/simulations/questions?externalIds=APPSC-M1-001,APPSC-M1-002
```

**Response (200):** `[{ "id": "uuid", "externalId": "APPSC-M1-001" }, …]` — only matches
are returned, and an empty/omitted `externalIds` returns `[]`.

#### POST /cms/simulations/questions/bulk

**No `examIds`** — questions inherit their scope from the simulation they are assigned into.
**1–500 items** (not 100). Duplicates are skipped on `externalId`, so a re-run is safe
rather than a failure.

| Field                                                            | Required | Notes                                                        |
| ---------------------------------------------------------------- | -------- | ------------------------------------------------------------ |
| `subject`, `difficulty`, `question`, `optionA`–`optionD`, `correctAnswer` | yes | `correctAnswer` is `A`/`B`/`C`/`D`, upper-cased server-side  |
| `externalId`                                                     | no       | The idempotency key for re-imports                            |
| `topic`, `microTopic`, `questionType`, `explanation`             | no       |                                                               |
| `pattern`                                                        | no       | Normalised server-side: `multi-stmt` → `MULTI_STATEMENT`, `assertion-reason` → `ASSERTION_REASONING`, otherwise upper-cased with `-`/space → `_` |
| `isCurrentAffairs`                                               | no       | default `false`                                               |
| `tags`                                                           | no       | default `[]`                                                  |
| `isVerified`                                                     | no       | default `false`                                               |
| `isActive`                                                       | no       | default `true`                                                |

**Response (201):** `{ "count": 148, "skipped": 2 }` — `skipped` is the items that already
existed under the same `externalId`.

#### POST /cms/simulations/:id/assign-questions

**Replaces** the whole list. Order is the array index (1-based `position`), and
`totalQuestions` is set to the array length. Transactional.

```json
{ "questionIds": ["7a1e…", "8b2f…", "9c30…"] }
```

**Response (200):** `{ "simulationId": "…", "totalQuestions": 3, "assigned": 3 }`

**Errors:**

| Status | Message |
| ------ | ------- |
| `400`  | `Question IDs not found: <ids>` |
| `400`  | `Duplicate question IDs in payload: <ids>` — the link table has a composite PK |
| `404`  | `Simulation <id> not found` |

Sending a shorter list **unassigns** the questions you left out. They are not deleted — they
go back to being unassigned bank rows.

---

### Psychometric Questions — `/cms/psychometric/questions`

| Method   | Endpoint                           | Description                           |
| -------- | ---------------------------------- | ------------------------------------- |
| `POST`   | `/cms/psychometric/questions`      | Create single question                |
| `POST`   | `/cms/psychometric/questions/bulk` | Bulk create (max 100, all-or-nothing) |
| `GET`    | `/cms/psychometric/questions`      | List with filters + pagination        |
| `GET`    | `/cms/psychometric/questions/:id`  | Get single by UUID                    |
| `PATCH`  | `/cms/psychometric/questions/:id`  | Update (partial)                      |
| `DELETE` | `/cms/psychometric/questions/:id`  | Hard delete                           |

---

#### Psychometric and the exam dimension

**Psychometric questions and test sets carry no `examIds`.** The column does not exist on
either model, so there is nothing to tag and nothing to mis-tag — every exam draws from the
same pool. An exam opts out of the personality test entirely with the per-exam
**`psychometric` feature flag**, not by withholding content: see
[SME_EXAMS_API.md](./SME_EXAMS_API.md) §7. Turning it off 403s the user-facing psychometric
routes for that exam while leaving these authoring routes open, which is why an SME can keep
editing test sets for an exam that does not offer the test.

This is also why psychometric is not mirrored on `/sme/content/*` (§0).

#### POST /cms/psychometric/questions

Create a new Psychometric question. Body as shown in Quick Start above.

#### POST /cms/psychometric/questions/bulk

Bulk create up to 100 Psychometric questions. Body: `{ "items": [...] }`.

#### GET /cms/psychometric/questions

List Psychometric questions with pagination and filters.

**Query params:**

| Param              | Type    | Default | Description                                    |
| ------------------ | ------- | ------- | ---------------------------------------------- |
| `page`             | integer | 1       | Page number (min: 1)                           |
| `limit`            | integer | 10      | Items per page (min: 1, max: 100)              |
| `subject`          | string  | —       | Filter by subject (case-insensitive)           |
| `topic`            | string  | —       | Filter by topic (case-insensitive)             |
| `psychometricType` | string  | —       | Filter by psychometric type                    |
| `difficulty`       | enum    | —       | `EASY`, `MEDIUM`, `HARD`, `EXPERT`             |
| `search`           | string  | —       | Search within question text (case-insensitive) |
| `isActive`         | boolean | —       | Filter by active status (`true`/`false`)       |
| `isVerified`       | boolean | —       | Filter by verified status (`true`/`false`)     |

**Request Examples:**

```
GET /cms/psychometric/questions?page=1&limit=20
GET /cms/psychometric/questions?psychometricType=COGNITIVE&difficulty=MEDIUM
GET /cms/psychometric/questions?topic=Logical Reasoning&isVerified=false
```

#### GET /cms/psychometric/questions/:id

Get full Psychometric question by UUID. Returns the complete question object.

**Errors:** `404` — Resource not found

#### PATCH /cms/psychometric/questions/:id

Update a Psychometric question. Send only the fields you want to change.

```json
{
  "isVerified": true,
  "correctAnswer": "C"
}
```

**Errors:** `404` — Resource not found, `400` — Validation failed

#### DELETE /cms/psychometric/questions/:id

Hard delete — permanently removes the question from the database.

**Errors:** `404` — Resource not found

---

### Psychometric Test Sets — `/cms/psychometric/test-sets`

| Method   | Endpoint                          | Description              |
| -------- | --------------------------------- | ------------------------ |
| `POST`   | `/cms/psychometric/test-sets`     | Create test set          |
| `GET`    | `/cms/psychometric/test-sets`     | List all (paginated)     |
| `GET`    | `/cms/psychometric/test-sets/:id` | Get test set + questions |
| `PATCH`  | `/cms/psychometric/test-sets/:id` | Update test set          |
| `DELETE` | `/cms/psychometric/test-sets/:id` | Hard delete              |

---

#### POST /cms/psychometric/test-sets

Create a new test set by grouping existing Psychometric question IDs. Body as shown in Quick Start above.

#### GET /cms/psychometric/test-sets

List all test sets with pagination.

**Query params:**

| Param   | Type    | Default | Description                       |
| ------- | ------- | ------- | --------------------------------- |
| `page`  | integer | 1       | Page number (min: 1)              |
| `limit` | integer | 10      | Items per page (min: 1, max: 100) |

**Request Examples:**

```
GET /cms/psychometric/test-sets?page=1&limit=10
```

#### GET /cms/psychometric/test-sets/:id

Get a single test set by UUID. Response includes the full list of associated questions.

**Errors:** `404` — Resource not found

#### PATCH /cms/psychometric/test-sets/:id

Update a test set. Send only the fields you want to change.

```json
{
  "name": "Logical Reasoning — Set A (Revised)",
  "questionIds": ["uuid-1", "uuid-2", "uuid-3", "uuid-4"]
}
```

**Errors:** `404` — Resource not found, `400` — Validation failed

#### DELETE /cms/psychometric/test-sets/:id

Hard delete — permanently removes the test set from the database.

**Errors:** `404` — Resource not found

---

## Field Reference

### PYQ Question Fields

| Field               | Type     | Required | Default | Notes                                            |
| ------------------- | -------- | -------- | ------- | ------------------------------------------------ |
| `examIds`           | string[] | no       | `["upsc-cse"]` | Exam modes this row belongs to. `["*"]` = every exam. Trimmed / lower-cased / de-duplicated. **Required** on `POST /sme/content/pyq` |
| `examName`          | string   | yes      | —       | e.g., "UPSC CSE", "UPSC CDS"                     |
| `year`              | integer  | yes      | —       | Exam year                                        |
| `paperNumber`       | integer  | no       | —       | Paper number (1, 2, etc.)                        |
| `subject`           | string   | yes      | —       | e.g., "Polity", "Geography", "History"           |
| `topic`             | string   | no       | —       | Specific topic within subject                    |
| `question`          | string   | yes      | —       | Full question text                               |
| `optionA`           | string   | yes      | —       | Option A text                                    |
| `optionB`           | string   | yes      | —       | Option B text                                    |
| `optionC`           | string   | no       | —       | Option C text                                    |
| `optionD`           | string   | no       | —       | Option D text                                    |
| `optionE`           | string   | no       | —       | Option E text (for 5-option questions)           |
| `correctAnswer`     | string   | yes      | —       | Single letter (A-E)                              |
| `answerExplanation` | string   | no       | —       | Detailed explanation of the correct answer       |
| `solveTip`          | string   | no       | —       | Quick solving strategy or memory aid             |
| `difficulty`        | enum     | no       | —       | `EASY`, `MEDIUM`, `HARD`, `EXPERT`               |
| `source`            | string   | no       | —       | Source reference                                 |
| `nature`            | string   | no       | —       | Question nature/type                             |
| `questionNumber`    | integer  | no       | —       | Position in paper                                |
| `totalMarks`        | integer  | no       | —       | Marks for the question                           |
| `duration`          | integer  | no       | —       | Expected time in seconds                         |
| `markingScheme`     | string   | no       | —       | e.g., "+2/-0.66"                                 |
| `tags`              | string[] | no       | `[]`    | Tags for categorization                          |
| `keywords`          | string[] | no       | `[]`    | Search keywords                                  |
| `relatedTopics`     | string[] | no       | `[]`    | Cross-referenced topics                          |
| `prediction`        | string   | no       | —       | Prediction score/label                           |
| `weightage`         | float    | no       | —       | Topic weightage                                  |
| `isActive`          | boolean  | no       | `true`  | Visibility flag — set `false` to hide from users |
| `isVerified`        | boolean  | no       | `false` | Review/QA flag — set `true` after verification   |

> ⚠️ **The Required column above understates it.** The create DTO is generated from
> `prisma/schema.prisma` (`pyq_papers`), where `difficulty`, `source`, `nature`,
> `totalMarks` and `duration` are all NOT NULL with no default — so they are **required** and
> a body omitting any of them is a `400`. Conversely `optionA`/`optionB` are nullable in the
> model and are **not** enforced. `source` ∈ `NCERT_STANDARD_REFERENCE_BOOK` /  `PYQ_THEME` /
> `CURRENT_AFFAIRS` / `UNCONVENTIONAL_SOURCE`; `nature` ∈ `CORE` / `CORE_PLUS` / `NEWS` /
> `NEWS_PLUS` / `WILD_CARD`. Some of the curl examples below predate this note; the create
> ones have been corrected.

### Mains Question Fields

| Field               | Type     | Required | Default | Notes                                    |
| ------------------- | -------- | -------- | ------- | ---------------------------------------- |
| `examIds`           | string[] | no       | `["upsc-cse"]` | Exam modes this row belongs to. `["*"]` = every exam. **Required** on `POST /sme/content/mains` |
| `examName`          | string   | yes      | —       | e.g., "UPSC CSE Mains"                   |
| `year`              | integer  | yes      | —       | Exam year                                |
| `subject`           | string   | yes      | —       | e.g., "GS Paper 2", "Essay"              |
| `topic`             | string   | no       | —       | Specific topic within subject            |
| `question`          | string   | yes      | —       | Full question text                       |
| `answer`            | string   | no       | —       | Model answer text                        |
| `answerExplanation` | string   | no       | —       | Additional explanation or approach notes |
| `solveTip`          | string   | no       | —       | Answer structuring tip                   |
| `difficulty`        | enum     | no       | —       | `EASY`, `MEDIUM`, `HARD`, `EXPERT`       |
| `questionNumber`    | integer  | no       | —       | Position in paper                        |
| `prediction`        | string   | no       | —       | Prediction score/label                   |
| `tags`              | string[] | no       | `[]`    | Tags for categorization                  |
| `keywords`          | string[] | no       | `[]`    | Search keywords                          |
| `relatedTopics`     | string[] | no       | `[]`    | Cross-referenced topics                  |
| `isActive`          | boolean  | no       | `true`  | Visibility flag                          |
| `isVerified`        | boolean  | no       | `false` | Review/QA flag                           |
| `marks`             | integer  | no       | `10`    | Marks for the question                   |
| `isOptional`        | boolean  | no       | derived | Optional-subject paper. Derived on **create** from a `" Optional"` suffix on `subject` when omitted; never re-derived on `PATCH` |

> ⚠️ Same caveat as PYQ: `difficulty` is NOT NULL with no default in
> `prisma/schema.prisma` (`mains_papers`), so it is **required**. `answer` is nullable — a
> question can be loaded before its model answer exists.

### Simulation Fields

See the [Simulations section](#simulations-mock-tests--cmssimulations) above for the create /
update field table and the container-carries-the-exam model.

### Psychometric Question Fields

| Field              | Type     | Required | Default | Notes                              |
| ------------------ | -------- | -------- | ------- | ---------------------------------- |
| `subject`          | string   | yes      | —       | e.g., "Aptitude"                   |
| `topic`            | string   | yes      | —       | e.g., "Logical Reasoning"          |
| `psychometricType` | string   | yes      | —       | e.g., "COGNITIVE"                  |
| `questionNumber`   | integer  | no       | —       | Position in set                    |
| `question`         | string   | yes      | —       | Full question text                 |
| `optionA`          | string   | yes      | —       | Option A text                      |
| `optionB`          | string   | yes      | —       | Option B text                      |
| `optionC`          | string   | no       | —       | Option C text                      |
| `optionD`          | string   | no       | —       | Option D text                      |
| `correctAnswer`    | string   | yes      | —       | Single letter (A-D)                |
| `difficulty`       | enum     | no       | —       | `EASY`, `MEDIUM`, `HARD`, `EXPERT` |
| `tags`             | string[] | no       | `[]`    | Tags for categorization            |
| `keywords`         | string[] | no       | `[]`    | Search keywords                    |
| `isActive`         | boolean  | no       | `true`  | Visibility flag                    |
| `isVerified`       | boolean  | no       | `false` | Review/QA flag                     |

> **No `examIds`.** Psychometric content has no exam dimension — see
> [Psychometric and the exam dimension](#psychometric-and-the-exam-dimension).

### Test Set Fields

| Field         | Type     | Required | Default | Notes                                |
| ------------- | -------- | -------- | ------- | ------------------------------------ |
| `name`        | string   | yes      | —       | Display name for the test set        |
| `description` | string   | no       | —       | Brief description of the test set    |
| `questionIds` | string[] | yes      | —       | Array of Psychometric question UUIDs |
| `isActive`    | boolean  | no       | `true`  | Visibility flag                      |

---

## Curl Examples

Every create/update example below carries `examIds`. **Omitting it is legal on `/cms/*` and
means `["upsc-cse"]`** — which is the whole trap §0 is about, so the examples always say it
out loud. The psychometric examples have no `examIds` because that model has none.

### PYQ — Create Single (tagged to one exam)

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/pyq \
  -H "Content-Type: application/json" \
  -d '{
    "examIds": ["appsc-group-1"],
    "examName": "APPSC Group 1",
    "year": 2024,
    "paperNumber": 1,
    "subject": "History",
    "topic": "Modern India",
    "difficulty": "EASY",
    "source": "PYQ_THEME",
    "nature": "CORE",
    "question": "Who founded the Indian National Congress?",
    "optionA": "Mahatma Gandhi",
    "optionB": "A.O. Hume",
    "optionC": "Jawaharlal Nehru",
    "optionD": "Bal Gangadhar Tilak",
    "correctAnswer": "B",
    "answerExplanation": "Allan Octavian Hume founded the INC in 1885.",
    "totalMarks": 2,
    "duration": 72
  }'
```

### PYQ — List with Filters (scoped to an exam)

```bash
curl "{{BASE_URL}}/api/v1/cms/pyq?exam=appsc-group-1&subject=History&year=2024&difficulty=EASY&page=1&limit=10"
```

### PYQ — Re-tag a row that landed in the wrong catalogue

```bash
curl -X PATCH {{BASE_URL}}/api/v1/cms/pyq/3f6c0e2a-0000-0000-0000-000000000000 \
  -H "Content-Type: application/json" \
  -d '{ "examIds": ["appsc-group-1"] }'
```

### PYQ — Bulk Create (mixed exams, including the `*` sentinel)

`examIds` is per item. `["*"]` means **every exam, including ones that do not exist yet** —
use it for material that is genuinely identical across exams, so a new exam inherits it with
no re-tagging.

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/pyq/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      {
        "examIds": ["appsc-group-1"],
        "examName": "APPSC Group 1",
        "year": 2024,
        "paperNumber": 1,
        "subject": "History",
        "topic": "Ancient India",
        "difficulty": "EASY",
        "source": "PYQ_THEME",
        "nature": "CORE",
        "question": "The Indus Valley Civilization was primarily located in?",
        "optionA": "Ganges Basin",
        "optionB": "Indus-Ghaggar-Hakra river system",
        "optionC": "Deccan Plateau",
        "optionD": "Brahmaputra Valley",
        "correctAnswer": "B",
        "totalMarks": 2,
        "duration": 72
      },
      {
        "examIds": ["*"],
        "examName": "UPSC CSE",
        "year": 2024,
        "paperNumber": 1,
        "subject": "History",
        "topic": "Ancient India",
        "difficulty": "EASY",
        "source": "PYQ_THEME",
        "nature": "CORE",
        "question": "Mohenjo-daro is located in present-day?",
        "optionA": "India",
        "optionB": "Pakistan",
        "optionC": "Afghanistan",
        "optionD": "Bangladesh",
        "correctAnswer": "B",
        "totalMarks": 2,
        "duration": 72
      }
    ]
  }'
```

### Mains — Create Single

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/mains \
  -H "Content-Type: application/json" \
  -d '{
    "examIds": ["upsc-cse"],
    "examName": "UPSC CSE Mains",
    "year": 2024,
    "subject": "GS Paper 1",
    "topic": "Indian Society",
    "question": "Examine the role of caste in Indian politics and its impact on democratic governance.",
    "answer": "Caste has been a significant factor in Indian politics since independence...",
    "difficulty": "HARD",
    "tags": ["caste", "politics", "democracy"]
  }'
```

### Mains — List with Filters

```bash
curl "{{BASE_URL}}/api/v1/cms/mains?exam=upsc-cse&subject=GS Paper 1&year=2024&page=1&limit=10"
```

### Mains — Bulk Create

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/mains/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      {
        "examIds": ["upsc-cse"],
        "examName": "UPSC CSE Mains",
        "year": 2024,
        "subject": "GS Paper 2",
        "topic": "Governance",
        "question": "Discuss the role of civil services in policy implementation.",
        "difficulty": "MEDIUM"
      },
      {
        "examIds": ["upsc-cse"],
        "examName": "UPSC CSE Mains",
        "year": 2024,
        "subject": "GS Paper 2",
        "topic": "Governance",
        "question": "Evaluate the impact of RTI Act on transparency in governance.",
        "difficulty": "MEDIUM"
      }
    ]
  }'
```

### Simulation — Create, load questions, assign (the full sequence)

```bash
# 1. the container carries the exam
curl -X POST {{BASE_URL}}/api/v1/cms/simulations \
  -H "Content-Type: application/json" \
  -d '{
    "examIds": ["appsc-group-1"],
    "name": "APPSC Group 1 — Full Mock 1",
    "durationMinutes": 150,
    "correctMark": 1.0,
    "wrongMark": -0.33,
    "sortOrder": 1
  }'

# 2. the question bank has NO examIds — scope comes from the simulation
curl -X POST {{BASE_URL}}/api/v1/cms/simulations/questions/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "externalId": "APPSC-M1-001", "subject": "Polity", "difficulty": "MEDIUM",
        "pattern": "multi-stmt",
        "question": "Consider the following statements…",
        "optionA": "1 only", "optionB": "2 only", "optionC": "Both", "optionD": "Neither",
        "correctAnswer": "C" }
    ]
  }'

# 3. resolve externalIds → UUIDs
curl "{{BASE_URL}}/api/v1/cms/simulations/questions?externalIds=APPSC-M1-001"

# 4. assign — this is what puts the questions into the exam
curl -X POST {{BASE_URL}}/api/v1/cms/simulations/b0ac5f18-0000-0000-0000-000000000000/assign-questions \
  -H "Content-Type: application/json" \
  -d '{ "questionIds": ["7a1e0000-0000-0000-0000-000000000000"] }'
```

### Simulation — List, scoped to an exam

```bash
curl "{{BASE_URL}}/api/v1/cms/simulations?exam=appsc-group-1"
```

### Psychometric Question — Create Single

> No `examIds` in the three psychometric examples below, and none in the test-set ones:
> that model has no exam column. See
> [Psychometric and the exam dimension](#psychometric-and-the-exam-dimension).

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/psychometric/questions \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Aptitude",
    "topic": "Verbal Reasoning",
    "psychometricType": "COGNITIVE",
    "question": "Choose the word most similar in meaning to: Ephemeral",
    "optionA": "Permanent",
    "optionB": "Transient",
    "optionC": "Solid",
    "optionD": "Ancient",
    "correctAnswer": "B",
    "difficulty": "MEDIUM"
  }'
```

### Psychometric Question — List with Filters

```bash
curl "{{BASE_URL}}/api/v1/cms/psychometric/questions?psychometricType=COGNITIVE&topic=Verbal%20Reasoning&page=1&limit=10"
```

### Psychometric Question — Bulk Create

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/psychometric/questions/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      {
        "subject": "Aptitude",
        "topic": "Numerical Ability",
        "psychometricType": "COGNITIVE",
        "question": "What is 15% of 240?",
        "optionA": "24",
        "optionB": "36",
        "optionC": "30",
        "optionD": "40",
        "correctAnswer": "B",
        "difficulty": "EASY"
      },
      {
        "subject": "Aptitude",
        "topic": "Numerical Ability",
        "psychometricType": "COGNITIVE",
        "question": "If x + 3 = 7, what is x?",
        "optionA": "3",
        "optionB": "4",
        "optionC": "5",
        "optionD": "10",
        "correctAnswer": "B",
        "difficulty": "EASY"
      }
    ]
  }'
```

### Test Set — Create

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/psychometric/test-sets \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Cognitive Ability — Beginner Set",
    "description": "Entry-level cognitive assessment with 10 questions",
    "questionIds": [
      "question-uuid-1",
      "question-uuid-2",
      "question-uuid-3"
    ]
  }'
```

### Test Set — List

```bash
curl "{{BASE_URL}}/api/v1/cms/psychometric/test-sets?page=1&limit=10"
```

---

## Workflow for SME/AI Stack

0. **Decide the exam tag before you start.** Every PYQ / Mains / Simulation row needs
   `examIds`. A forgotten tag is not an error here — it is UPSC content (§0). New
   integrations should do all of this against
   [`/sme/content/*`](./SME_CONTENT_INGEST_API.md), where the tag is mandatory.
1. **Populate questions:** Use bulk create endpoints to upload batches of PYQ, Mains, or Psychometric questions (up to 100 per request), carrying `examIds` on every item
2. **Verify the tags landed:** `GET /sme/exams/:id/content-counts` — `exclusive: 0` on a
   module you just loaded into means the tags did not land. Do this before anything else;
   a mis-tagged load gets harder to unpick the longer it sits.
3. **Review:** List questions with filters (`?exam=<slug>&isVerified=false`) to verify data quality
4. **Verify:** `PATCH` individual questions to set `isVerified: true` after review
5. **Organize (Psychometric):** Create test sets by grouping verified Psychometric question UUIDs
6. **Organize (Simulations):** create the simulation → bulk-create its questions → `assign-questions`
7. **Deactivate:** `PATCH` with `{ "isActive": false }` to hide content from users without deleting
8. **Clean up:** Use `DELETE` only for permanently removing incorrect or duplicate entries
   (note `DELETE /cms/simulations/:id` is a **soft** delete, unlike PYQ and Mains)

---

## Common Errors

| Status | Cause                         | Fix                                                                      |
| ------ | ----------------------------- | ------------------------------------------------------------------------ |
| 400    | Missing required field        | Check the Field Reference tables above for required fields               |
| 400    | Unique constraint violation   | A record with the same unique key combination already exists             |
| 400    | Bulk create exceeds 100 items | Split into multiple requests of 100 or fewer (simulation *questions* allow 500) |
| 400    | `Unknown exam "<slug>". Create it via POST /sme/exams first.` | Only from **`?exam=`** on a list call. A bad slug in a create **body** never 400s here — it is either stored verbatim (if well-formed) or dropped to `["upsc-cse"]` (if malformed) |
| 404    | Resource not found            | Verify the UUID is correct and the resource exists                       |
| 422    | Invalid UUID format           | Ensure `:id` params are valid UUIDs                                      |
| 422    | Invalid enum value            | Use exact enum values: `EASY`, `MEDIUM`, `HARD`, `EXPERT` for difficulty |

### Error Response Format

```json
{
  "statusCode": 400,
  "message": "Validation failed: examName should not be empty",
  "error": "Bad Request"
}
```

---

## Quick Reference — All Endpoints

| Method   | Endpoint                           | Description                                       |
| -------- | ---------------------------------- | ------------------------------------------------- |
| `POST`   | `/cms/pyq`                         | Create PYQ question                               |
| `POST`   | `/cms/pyq/bulk`                    | Bulk create PYQ questions                         |
| `GET`    | `/cms/pyq`                         | List PYQ questions (filtered, paginated)          |
| `GET`    | `/cms/pyq/:id`                     | Get single PYQ question                           |
| `PATCH`  | `/cms/pyq/:id`                     | Update PYQ question                               |
| `DELETE` | `/cms/pyq/:id`                     | Delete PYQ question                               |
| `POST`   | `/cms/mains`                       | Create Mains question                             |
| `POST`   | `/cms/mains/bulk`                  | Bulk create Mains questions                       |
| `GET`    | `/cms/mains`                       | List Mains questions (filtered, paginated)        |
| `GET`    | `/cms/mains/:id`                   | Get single Mains question                         |
| `PATCH`  | `/cms/mains/:id`                   | Update Mains question                             |
| `DELETE` | `/cms/mains/:id`                   | Delete Mains question                             |
| `GET`    | `/cms/simulations`                 | List all simulations incl. inactive (`?exam=`)    |
| `POST`   | `/cms/simulations`                 | Create simulation                                 |
| `PATCH`  | `/cms/simulations/:id`             | Update simulation                                 |
| `DELETE` | `/cms/simulations/:id`             | **Soft** delete (sets `isActive: false`)          |
| `GET`    | `/cms/simulations/questions`       | Resolve `externalId` → UUID                       |
| `POST`   | `/cms/simulations/questions/bulk`  | Bulk create simulation questions (1–500)          |
| `POST`   | `/cms/simulations/:id/assign-questions` | Replace the assigned question list           |
| `POST`   | `/cms/psychometric/questions`      | Create Psychometric question                      |
| `POST`   | `/cms/psychometric/questions/bulk` | Bulk create Psychometric questions                |
| `GET`    | `/cms/psychometric/questions`      | List Psychometric questions (filtered, paginated) |
| `GET`    | `/cms/psychometric/questions/:id`  | Get single Psychometric question                  |
| `PATCH`  | `/cms/psychometric/questions/:id`  | Update Psychometric question                      |
| `DELETE` | `/cms/psychometric/questions/:id`  | Delete Psychometric question                      |
| `POST`   | `/cms/psychometric/test-sets`      | Create test set                                   |
| `GET`    | `/cms/psychometric/test-sets`      | List test sets (paginated)                        |
| `GET`    | `/cms/psychometric/test-sets/:id`  | Get test set + questions                          |
| `PATCH`  | `/cms/psychometric/test-sets/:id`  | Update test set                                   |
| `DELETE` | `/cms/psychometric/test-sets/:id`  | Delete test set                                   |

### Common Filter Parameters (PYQ, Mains, Psychometric Questions)

| Parameter          | Type    | Description                                 |
| ------------------ | ------- | ------------------------------------------- |
| `exam`             | string  | Exam mode slug (PYQ, Mains, Simulations only). Matches that exam **or** `*`. Omit for every exam. **Unknown slug = 400**, never an empty page |
| `page`             | integer | Page number (default: 1)                    |
| `limit`            | integer | Items per page (default: 10, max: 100)      |
| `subject`          | string  | Case-insensitive filter                     |
| `year`             | integer | Exact match (PYQ, Mains only)               |
| `topic`            | string  | Case-insensitive filter (Psychometric only) |
| `psychometricType` | string  | Filter by type (Psychometric only)          |
| `difficulty`       | enum    | `EASY` / `MEDIUM` / `HARD` / `EXPERT`       |
| `search`           | string  | Search in question text (case-insensitive)  |
| `isActive`         | boolean | `true` or `false`                           |
| `isVerified`       | boolean | `true` or `false`                           |
