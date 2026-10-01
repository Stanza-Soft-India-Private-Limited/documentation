# Content Document — SME / AI Stack API Guide

> Changed 2026-09-21 — exam-safe ingest: /sme/content/* + examIds on /cms and content-doc (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).
> Changed 2026-09-21 — per-exam broadcast, validated segment exam (+ fix), survey targetExams, feedback ?exam= (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

## Overview

This API allows you to create, update, list, and manage the library's study documents. You send raw markdown (with embedded quiz blocks) and the backend automatically parses it into structured sections (content, images, quizzes) for the mobile app.

**Base URL:** `{API_BASE_URL}/api/v1/content-doc-admin`
**Authentication:** None required today — these routes are `@Public()`, exactly like `/cms/*`. There is no `x-api-key` on this surface.
**Note:** All endpoints use the `/api/v1/` global prefix.

**The platform is multi-exam.** Every document carries an `examIds` array (UPSC / APPSC /
TPSC / …). It is **optional and defaults to `["upsc-cse"]`** — see
[Exam tagging](#exam-tagging-examids) below.

---

## Quick Start

### Create a Document

```
POST /content-doc-admin
Content-Type: application/json
```

**Request Body:**

```json
{
  "title": "Fundamental Rights - Part III",
  "subject": "Polity",
  "topic": "Fundamental Rights",
  "subTopic": "Right to Equality",
  "rawMarkdown": "## Introduction\n\nThe Fundamental Rights are...\n\n```quiz\nQ: How many FR exist?\nA) Five\nB) Six\nC) Seven\nD) Eight\nCorrect: B\nExplanation: After 44th Amendment, six FRs remain.\n```\n\n## Summary\n\nFRs are the cornerstone...",
  "status": "DRAFT",
  "language": "ENGLISH",
  "examIds": ["*"]
}
```

**Required fields:** `title`, `subject`, `topic`, `rawMarkdown`
**Optional fields:** `subTopic`, `status` (DRAFT | PUBLISHED | ARCHIVED, default: DRAFT), `language` (default: ENGLISH), `sourceId`, `examIds` (default: `["upsc-cse"]`)

**Response (201):**
Returns the created document with:
- `id` — UUID (use this for updates/retrieval)
- `slug` — auto-generated URL slug
- `sections` — parsed JSON array of blocks (with question UUIDs)
- `totalQuestions` — count of quiz questions parsed
- `estimatedReadMinutes` — word count / 200 wpm
- `questions` — array of all ContentQuestion records created
- `examIds` — the **normalised** tags actually stored (read this back, don't assume)

**Typical execution time:** ~200-500ms for a standard document (2000-5000 words, 5-15 questions).

---

## Exam tagging (`examIds`)

| You send | Stored | Notes |
|---|---|---|
| `["appsc-group-1"]` | `["appsc-group-1"]` | this exam's library only |
| `[" APPSC-Group-1 "]` | `["appsc-group-1"]` | trimmed + lower-cased |
| `["upsc-cse", "UPSC-CSE"]` | `["upsc-cse"]` | de-duplicated |
| `["*"]` | `["*"]` | **every exam**, including ones that do not exist yet |
| `["appsc-grp-1"]` — well-formed, but not a real exam | `["appsc-grp-1"]` + a server WARN | ⚠️ stored verbatim and matches nothing anywhere. Since 2026-09-21 the catalogue **is** consulted and an unknown slug logs a warning — but it is a **log line only**: still no 400, still `201`, nothing the client can read |
| `["APPSC Group 1"]` — not a valid slug shape | `["upsc-cse"]` | values failing `^[a-z0-9]+(-[a-z0-9]+)*$` are dropped, not rejected |
| omitted | `["upsc-cse"]` | this is what keeps every existing caller unchanged |

`*` is a sentinel, not a wildcard expansion: reads match on
`examIds hasSome [<exam>, "*"]`, so `*`-tagged documents are visible to every exam and never
need re-tagging when a new one launches. **Core GS study material is usually `["*"]`;
state-specific material is not.**

**On `PATCH`, an omitted `examIds` leaves the column alone.** Normalisation runs only when
the key is present in the body, so a `{"status": "PUBLISHED"}` edit can never re-tag the
document.

⚠️ **A wrong tag is silent on the wire.** A document tagged `["upsc-cse"]` by accident is
invisible in the APPSC library and shows up in the UPSC one, with no error anywhere. After a
load, check `GET /sme/exams/:id/content-counts` and read the **`library`** row's `exclusive`
count — zero means the tags did not land. See [SME_EXAMS_API.md](./SME_EXAMS_API.md) §2.

**Server-side WARNs (added 2026-09-21), for whoever can read the logs during a load.** Create
and update log a warning when a slug is well-formed but names no exam:

> *"Content document tagged with exam "appsc-grp-1", which is not in the catalogue — the
> content is being stored with that tag anyway and will be invisible to every exam. Check the
> slug, or create the exam via POST /sme/exams."*

On update the check runs **only when the caller actually sent `examIds`**, so an omitted key
is never re-checked or re-defaulted. `*` is skipped (it is not a catalogue entry). This is a
tripwire, **never a rejection** — content-doc admin is an existing script surface and has to
keep working, so the row is written with the bad tag either way.

Unlike PYQ / Mains / Simulations, there is **no key-gated mirror of this route** that makes
`examIds` mandatory — `/sme/content/*` covers the three question banks only
([SME_CONTENT_INGEST_API.md](./SME_CONTENT_INGEST_API.md) §7).

---

## Markdown Format Specification

The markdown must follow this exact format. The parser is strict about the quiz block syntax.

### Text Content
Standard markdown — headings, bold, italic, lists, links, etc. All standard markdown is supported and passed through as-is to the mobile app's markdown renderer.

### Images
Standalone image lines (must be on their own line, not inline):
```markdown
![Alt text description](https://your-cdn.com/path/to/image.png)
```
- The image URL must be a full absolute URL
- Alt text is required (used for accessibility)
- Image must be on its own line (not inline with text)

### Quiz Blocks
Fenced with triple backticks and the `quiz` keyword:

````markdown
```quiz
## Optional Quiz Title

Q: Your question text here?
A) First option
B) Second option
C) Third option
D) Fourth option
Correct: B
Explanation: Why B is the correct answer.

Q: Another question?
A) Option A
B) Option B
Correct: A
Explanation: Reasoning here.
```
````

**Rules:**
- Quiz title line (`## Title`) is optional — if present, must be the first line inside the fence
- Each question starts with `Q:` on a new line
- Options use `A)` `B)` `C)` `D)` format — A and B are required, C and D are optional
- `Correct:` must be a single letter (A, B, C, or D)
- `Explanation:` is optional but strongly recommended
- Separate questions with a blank line
- You can have multiple quiz blocks throughout the document

---

## Complete Markdown Example

````markdown
## Introduction to Fundamental Rights

The Fundamental Rights are enshrined in **Part III** of the Indian Constitution (Articles 12-35). These rights are justiciable.

![Fundamental Rights Overview](https://cdn.prepmonkey.ai/images/fundamental-rights.png)

### Currently, there are six Fundamental Rights:

1. **Right to Equality** (Articles 14-18)
2. **Right to Freedom** (Articles 19-22)
3. **Right against Exploitation** (Articles 23-24)
4. **Right to Freedom of Religion** (Articles 25-28)
5. **Cultural and Educational Rights** (Articles 29-30)
6. **Right to Constitutional Remedies** (Article 32)

```quiz
## Test: Fundamental Rights Basics

Q: How many Fundamental Rights are currently recognized?
A) Five
B) Six
C) Seven
D) Eight
Correct: B
Explanation: After the removal of Right to Property by the 44th Amendment, there are now six.

Q: Which Part of the Constitution deals with Fundamental Rights?
A) Part II
B) Part III
C) Part IV
D) Part V
Correct: B
Explanation: Part III covers Articles 12 to 35.
```

## Right to Equality

Article 14 guarantees **equality before the law** and **equal protection of the laws**.

```quiz
Q: Article 14 guarantees which of the following?
A) Right to Freedom of Speech
B) Equality before law and equal protection of laws
C) Right against Exploitation
D) Right to Freedom of Religion
Correct: B
Explanation: Article 14 provides two concepts from British and American law respectively.
```

## Summary

The Fundamental Rights form the cornerstone of Indian democracy.
````

**This produces:** 7 sections (3 content + 1 image + 2 quiz + 1 content), 3 questions.

---

## All Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/content-doc-admin` | Create a document from raw markdown |
| `GET` | `/content-doc-admin` | List (paginated, filterable, `?exam=`) |
| `GET` | `/content-doc-admin/by-source/:sourceId` | Look up by the generator's source ID |
| `GET` | `/content-doc-admin/:id` | Get one, including `rawMarkdown` |
| `PATCH` | `/content-doc-admin/:id` | Update (re-parses if markdown changed) |
| `POST` | `/content-doc-admin/reparse-all` | Re-parse every document with `totalQuestions = 0` |
| `POST` | `/content-doc-admin/:id/reparse` | Re-parse one document from its stored markdown |
| `DELETE` | `/content-doc-admin/:id` | Soft delete (`isActive: false`) |

### POST /content-doc-admin
Create a new document. Body as shown above.

### GET /content-doc-admin
List documents with pagination and filters.

**Query params:**
| Param | Type | Description |
|-------|------|-------------|
| page | number | Page number (default: 1) |
| limit | number | Items per page (default: 10, max: 100) |
| exam | string | Exam mode slug. Matches that exam **or** `*`. Omit for every exam. **Unknown slug = 400**, never a silently empty page |
| subject | string | Filter by subject (case-insensitive contains) |
| topic | string | Filter by topic (case-insensitive contains) |
| status | string | DRAFT, PUBLISHED, or ARCHIVED |
| search | string | Search within title (case-insensitive) |
| isActive | boolean | Filter by active status |

**Example:** `GET /content-doc-admin?subject=Polity&status=DRAFT&page=1&limit=20`
**Example:** `GET /content-doc-admin?exam=appsc-group-1&status=PUBLISHED`

Every row carries its `examIds`, so a mis-tagged upload is visible here. The list uses an
allow-list `select`, so the rows are a summary — `rawMarkdown`, `sections` and `questions`
are **not** included; fetch one document by id for those.

### GET /content-doc-admin/by-source/:sourceId
Look up a document by the `sourceId` the generator system assigned it, rather than by our
UUID. `sourceId` is unique, so this returns a single document with its full
`questions` array — the same shape as `GET /content-doc-admin/:id`.

Use it to make an ingest idempotent: check `by-source` first, `PATCH` if it comes back,
`POST` if it 404s.

**Errors:** `404` — `Content document with sourceId with ID <sourceId> not found` (the
doubled "with" is the live string, not a typo here — the shared not-found helper appends
`with ID <identifier>` to whatever name the caller passed)

### GET /content-doc-admin/:id
Get full document by ID. Response includes `rawMarkdown` for re-editing.

### PATCH /content-doc-admin/:id
Update a document. Send only the fields you want to change.

**If `rawMarkdown` is included:** All existing questions are deleted and recreated from the new markdown. This is a full overwrite of content.

**If only metadata changes** (title, subject, topic, status, etc.): Questions are untouched.

**`examIds` is only touched if you send it** — an omitted key leaves the row's tags alone.

```json
{
  "status": "PUBLISHED"
}
```

```json
{
  "examIds": ["appsc-group-1"]
}
```

### POST /content-doc-admin/:id/reparse
Re-parse a document **from its existing `rawMarkdown`** — no body. Deletes and recreates its
`ContentQuestion` rows and rewrites `sections`, `totalQuestions` and `estimatedReadMinutes`.

This is the repair route for a document whose quiz fences were malformed when it was first
created: fix nothing, just re-run the parser after a parser fix. It does **not** change
`examIds`, `status` or any other metadata.

**Response (200):** the re-parsed document, `questions` included.
**Errors:** `404` — Content document not found

### POST /content-doc-admin/reparse-all
Re-parses **every document with `totalQuestions = 0`** — the signature of a broken quiz
fence. No body, no filters, and **no exam scoping**: it sweeps every exam's library at once.

```json
{
  "reparsed": 12,
  "documents": [
    { "id": "…", "title": "Fundamental Rights - Part III", "totalQuestions": 3 }
  ]
}
```

⚠️ It runs the documents **sequentially, each in its own transaction**, so a large sweep is
slow and is not all-or-nothing — a failure part-way leaves the earlier documents re-parsed.
A document that legitimately has no quiz blocks is re-parsed every time you call it, and
still comes back with `totalQuestions: 0`.

### DELETE /content-doc-admin/:id
Soft delete — sets `isActive: false`. Document is hidden from users but not destroyed.

---

## Workflow for SME/AI Stack

0. **Decide the exam tag first.** `["*"]` for shared GS material, the specific slug(s) for
   exam-specific material. Omitting it means `["upsc-cse"]`, silently.
1. **Draft phase:** `POST /content-doc-admin` with `status: "DRAFT"` (or omit status) and `examIds`
2. **Review:** `GET /content-doc-admin/:id` — check `totalQuestions`, `sections` structure, `estimatedReadMinutes`, `examIds`
3. **Revise:** `PATCH /content-doc-admin/:id` with updated `rawMarkdown` if needed
4. **Publish:** `PATCH /content-doc-admin/:id` with `{ "status": "PUBLISHED" }` — document becomes visible to app users
5. **Verify the tags landed:** `GET /sme/exams/:id/content-counts` → the `library` row's
   `exclusive` count. Zero after a load means the documents went to the wrong exam.
6. **Archive:** `PATCH /content-doc-admin/:id` with `{ "status": "ARCHIVED" }` — removes from user listing

---

## Common Errors

| Status | Cause | Fix |
|--------|-------|-----|
| 400 | Missing required field | Include `title`, `subject`, `topic`, `rawMarkdown` |
| 400 | Duplicate slug | Change the title slightly (slug is auto-generated) |
| 400 | `Unknown exam "<slug>". Create it via POST /sme/exams first.` | Only from **`?exam=`** on the list call. A bad slug in a create/update **body** never 400s — it is stored verbatim (if well-formed, now with a server WARN) or dropped to `["upsc-cse"]` (if malformed) |
| 404 | Invalid document ID | Check the UUID is correct |

---

## Tips for Writing Good Content

- Aim for **5-15 questions** per document for optimal engagement
- Place quiz blocks **after** the relevant content section (not all at the end)
- Keep documents under **3000 words** for mobile readability (~15 min read time)
- Use images to break up text-heavy sections
- Always provide explanations — they're shown to users after answering
