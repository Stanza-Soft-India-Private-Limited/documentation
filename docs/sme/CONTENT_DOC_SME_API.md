# Content Document — SME / AI Stack API Guide

## Overview

This API allows you to create, update, list, and manage study documents for the UPSC platform. You send raw markdown (with embedded quiz blocks) and the backend automatically parses it into structured sections (content, images, quizzes) for the mobile app.

**Base URL:** `{API_BASE_URL}/api/v1/content-doc-admin`
**Authentication:** None required (public CMS endpoints)
**Note:** All endpoints use the `/api/v1/` global prefix.

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
  "language": "ENGLISH"
}
```

**Required fields:** `title`, `subject`, `topic`, `rawMarkdown`
**Optional fields:** `subTopic`, `status` (DRAFT | PUBLISHED | ARCHIVED, default: DRAFT), `language` (default: ENGLISH)

**Response (201):**
Returns the created document with:
- `id` — UUID (use this for updates/retrieval)
- `slug` — auto-generated URL slug
- `sections` — parsed JSON array of blocks (with question UUIDs)
- `totalQuestions` — count of quiz questions parsed
- `estimatedReadMinutes` — word count / 200 wpm
- `questions` — array of all ContentQuestion records created

**Typical execution time:** ~200-500ms for a standard document (2000-5000 words, 5-15 questions).

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

### POST /content-doc-admin
Create a new document. Body as shown above.

### GET /content-doc-admin
List documents with pagination and filters.

**Query params:**
| Param | Type | Description |
|-------|------|-------------|
| page | number | Page number (default: 1) |
| limit | number | Items per page (default: 10, max: 100) |
| subject | string | Filter by subject (case-insensitive contains) |
| topic | string | Filter by topic (case-insensitive contains) |
| status | string | DRAFT, PUBLISHED, or ARCHIVED |
| search | string | Search within title (case-insensitive) |
| isActive | boolean | Filter by active status |

**Example:** `GET /content-doc-admin?subject=Polity&status=DRAFT&page=1&limit=20`

### GET /content-doc-admin/:id
Get full document by ID. Response includes `rawMarkdown` for re-editing.

### PATCH /content-doc-admin/:id
Update a document. Send only the fields you want to change.

**If `rawMarkdown` is included:** All existing questions are deleted and recreated from the new markdown. This is a full overwrite of content.

**If only metadata changes** (title, subject, topic, status, etc.): Questions are untouched.

```json
{
  "status": "PUBLISHED"
}
```

### DELETE /content-doc-admin/:id
Soft delete — sets `isActive: false`. Document is hidden from users but not destroyed.

---

## Workflow for SME/AI Stack

1. **Draft phase:** `POST /content-doc-admin` with `status: "DRAFT"` (or omit status)
2. **Review:** `GET /content-doc-admin/:id` — check `totalQuestions`, `sections` structure, `estimatedReadMinutes`
3. **Revise:** `PATCH /content-doc-admin/:id` with updated `rawMarkdown` if needed
4. **Publish:** `PATCH /content-doc-admin/:id` with `{ "status": "PUBLISHED" }` — document becomes visible to app users
5. **Archive:** `PATCH /content-doc-admin/:id` with `{ "status": "ARCHIVED" }` — removes from user listing

---

## Common Errors

| Status | Cause | Fix |
|--------|-------|-----|
| 400 | Missing required field | Include `title`, `subject`, `topic`, `rawMarkdown` |
| 400 | Duplicate slug | Change the title slightly (slug is auto-generated) |
| 404 | Invalid document ID | Check the UUID is correct |

---

## Tips for Writing Good Content

- Aim for **5-15 questions** per document for optimal engagement
- Place quiz blocks **after** the relevant content section (not all at the end)
- Keep documents under **3000 words** for mobile readability (~15 min read time)
- Use images to break up text-heavy sections
- Always provide explanations — they're shown to users after answering
