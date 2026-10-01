# PYQ mobile contract — figures, board-deleted questions, paper labels (2026-10)

What the app receives from the three PYQ read endpoints after the 2026-10 APPSC
fidelity work, and the one new error. **Every JSON block below is the real serialized
output of the backend** — `src/modules/pyq/pyq.mobile-contract.spec.ts` builds these
responses from fixtures and fails if any block here differs from what the code emits.
Regenerate with `PYQ_CONTRACT_DUMP=<file> npx jest pyq.mobile-contract` after an
intentional change.

All changes are **additive**. No key was removed or renamed; old builds that ignore
unknown keys (`ignoreUnknownKeys = true` in `HttpClientFactory`) keep working.

## What is new

| Field | Where | Type | Meaning |
|---|---|---|---|
| `paperLabel` | list row, details `question`, mock item | `String?` | Paper chip shown beside the year: `"PAPER-I"`, `"SET-A"`, `"MARCH"`. `null` for UPSC. |
| `questionImageUrl` | list row, details `question`, mock item | `String?` | Public URL of the figure in the question stem. `null` when there is none. Render it between the stem and the options. |
| `imageCaption` | list row, details `question`, mock item | `String?` | Caption under the figure. Can be `null` even when there is a figure. |
| `boardDeleted` | list row, details `question` | `Boolean` | The board withdrew this question after the exam. See below. |
| `examName` | details `question` (list rows already had it) | `String` | e.g. `"APPSC Group 1"`, `"UPSC CSE"`. Never null. |
| `paperNumber` | details `question` (list rows already had it) | `Int?` | `null` for UPSC. |

### Board-deleted questions (`boardDeleted: true`)

SHOWN but INERT. The question appears in the list, in its year folder and in details,
but:

- `correctAnswer`, `answerExplanation`/`explanation`, `solveTip`/`approach` are `null`
  for **every** tier, premium included.
- details: `canViewExplanation: false`, `unlockedToday: false`.
- **Never** in `GET /pyq/mock-test`.
- Excluded from `GET /pyq/metrics` totals and attempted counts (100 % is reachable).
- `POST /pyq/:id/submit`, `GET /pyq/:id/reveal` and **adding** a bookmark
  (`POST /pyq/:id/bookmark` on a question not yet bookmarked) answer **409
  `QUESTION_WITHDRAWN`** (body below). Reveal is refused **before** metering: no reveal
  credit is spent. Removing an existing bookmark still works (200).

Render it inert: show the stem and options, a "Deleted by the board" badge, no answer
buttons, no reveal, no bookmark-add.

### Options on details are never null

`question.optionA`–`optionD` on `GET /pyq/:id/details` / `/reveal` are **always
strings**. If the stored option is null (a statement-style / non-MCQ row, or an ingest
that left an option out) the server sends `""`. This matches `QuestionData`, which
declares A–D non-null. `optionE` stays `String?`. List rows and mock items are
unchanged: their options keep the column's null-ness (the app already models them as
`String?` there; mock items only ever come from rows that have `optionA`).

## 409 QUESTION_WITHDRAWN

Same body for submit, reveal and add-bookmark (only `path`/`method` differ). Key on
`code`, not on `message`.

```json contract=error.409
{
  "success": false,
  "message": "This question was withdrawn by the board and cannot be attempted.",
  "error": "Conflict",
  "code": "QUESTION_WITHDRAWN",
  "statusCode": 409,
  "timestamp": "<ISO-8601 timestamp>",
  "path": "/pyq/6f1c2a10-0000-4000-8000-000000000002/submit",
  "method": "POST"
}
```

## GET /pyq — list rows

`data[]` items. The row is the full database row plus `hasAttempted`,
`lastAttemptCorrect`, `isBookmarked`. Shown for a **premium** user; for a free user the
same row has `correctAnswer`, `answerExplanation`, `solveTip` set to `null` (case a2).

### (a) Live APPSC figure question — premium

```json contract=list.figure
{
  "id": "6f1c2a10-0000-4000-8000-000000000001",
  "examName": "APPSC Group 1",
  "year": 2024,
  "paperNumber": 1,
  "subject": "Geography",
  "topic": "Indian Rivers",
  "difficulty": "MEDIUM",
  "source": "OFFICIAL",
  "nature": "CONCEPTUAL",
  "questionNumber": 17,
  "question": "Which river is marked as X in the map given below?",
  "optionA": "Krishna",
  "optionB": "Godavari",
  "optionC": "Pennar",
  "optionD": "Vamsadhara",
  "optionE": null,
  "correctAnswer": "B",
  "answerExplanation": "The marked river rises near Nashik and enters AP near Bhadrachalam.",
  "solveTip": null,
  "prediction": null,
  "weightage": null,
  "repeatFrequency": 0,
  "totalMarks": 150,
  "duration": 150,
  "markingScheme": null,
  "tags": [],
  "keywords": [],
  "relatedTopics": [],
  "isActive": true,
  "isVerified": true,
  "language": "ENGLISH",
  "createdAt": "2026-10-01T06:30:00.000Z",
  "updatedAt": "2026-10-01T06:30:00.000Z",
  "examIds": [
    "appsc-group-1"
  ],
  "paperLabel": "PAPER-I",
  "questionImageUrl": "https://example-media-bucket.s3.ap-south-1.amazonaws.com/pyq-figures/3b1e9c2a-5d4f-4e8a-9c1b-7a2d6e0f4c11-appsc-2024-p1-q17.png",
  "imageCaption": "Map of Andhra Pradesh — rivers",
  "boardDeleted": false,
  "hasAttempted": false,
  "lastAttemptCorrect": null,
  "isBookmarked": false
}
```

### (a2) Same row — free user

```json contract=list.figure.free
{
  "id": "6f1c2a10-0000-4000-8000-000000000001",
  "examName": "APPSC Group 1",
  "year": 2024,
  "paperNumber": 1,
  "subject": "Geography",
  "topic": "Indian Rivers",
  "difficulty": "MEDIUM",
  "source": "OFFICIAL",
  "nature": "CONCEPTUAL",
  "questionNumber": 17,
  "question": "Which river is marked as X in the map given below?",
  "optionA": "Krishna",
  "optionB": "Godavari",
  "optionC": "Pennar",
  "optionD": "Vamsadhara",
  "optionE": null,
  "correctAnswer": null,
  "answerExplanation": null,
  "solveTip": null,
  "prediction": null,
  "weightage": null,
  "repeatFrequency": 0,
  "totalMarks": 150,
  "duration": 150,
  "markingScheme": null,
  "tags": [],
  "keywords": [],
  "relatedTopics": [],
  "isActive": true,
  "isVerified": true,
  "language": "ENGLISH",
  "createdAt": "2026-10-01T06:30:00.000Z",
  "updatedAt": "2026-10-01T06:30:00.000Z",
  "examIds": [
    "appsc-group-1"
  ],
  "paperLabel": "PAPER-I",
  "questionImageUrl": "https://example-media-bucket.s3.ap-south-1.amazonaws.com/pyq-figures/3b1e9c2a-5d4f-4e8a-9c1b-7a2d6e0f4c11-appsc-2024-p1-q17.png",
  "imageCaption": "Map of Andhra Pradesh — rivers",
  "boardDeleted": false,
  "hasAttempted": false,
  "lastAttemptCorrect": null,
  "isBookmarked": false
}
```

### (b) Board-deleted APPSC question (identical for every tier)

```json contract=list.deleted
{
  "id": "6f1c2a10-0000-4000-8000-000000000002",
  "examName": "APPSC Group 1",
  "year": 2024,
  "paperNumber": 1,
  "subject": "Economy",
  "topic": "Irrigation",
  "difficulty": "MEDIUM",
  "source": "OFFICIAL",
  "nature": "CONCEPTUAL",
  "questionNumber": 42,
  "question": "Consider the following statements about the Polavaram project …",
  "optionA": "1 only",
  "optionB": "2 only",
  "optionC": "Both 1 and 2",
  "optionD": "Neither 1 nor 2",
  "optionE": null,
  "correctAnswer": null,
  "answerExplanation": null,
  "solveTip": null,
  "prediction": null,
  "weightage": null,
  "repeatFrequency": 0,
  "totalMarks": 150,
  "duration": 150,
  "markingScheme": null,
  "tags": [],
  "keywords": [],
  "relatedTopics": [],
  "isActive": true,
  "isVerified": true,
  "language": "ENGLISH",
  "createdAt": "2026-10-01T06:30:00.000Z",
  "updatedAt": "2026-10-01T06:30:00.000Z",
  "examIds": [
    "appsc-group-1"
  ],
  "paperLabel": "PAPER-I",
  "questionImageUrl": null,
  "imageCaption": null,
  "boardDeleted": true,
  "hasAttempted": false,
  "lastAttemptCorrect": null,
  "isBookmarked": false
}
```

### (c) UPSC question — every new field null / false

```json contract=list.upsc
{
  "id": "6f1c2a10-0000-4000-8000-000000000003",
  "examName": "UPSC CSE",
  "year": 2023,
  "paperNumber": null,
  "subject": "Polity",
  "topic": "Fundamental Duties",
  "difficulty": "MEDIUM",
  "source": "OFFICIAL",
  "nature": "CONCEPTUAL",
  "questionNumber": 5,
  "question": "Which of the following is a Fundamental Duty?",
  "optionA": "To vote",
  "optionB": "To pay taxes",
  "optionC": "To protect the environment",
  "optionD": "To own property",
  "optionE": null,
  "correctAnswer": "C",
  "answerExplanation": "Article 51A(g).",
  "solveTip": null,
  "prediction": null,
  "weightage": null,
  "repeatFrequency": 0,
  "totalMarks": 200,
  "duration": 120,
  "markingScheme": null,
  "tags": [],
  "keywords": [],
  "relatedTopics": [],
  "isActive": true,
  "isVerified": true,
  "language": "ENGLISH",
  "createdAt": "2026-10-01T06:30:00.000Z",
  "updatedAt": "2026-10-01T06:30:00.000Z",
  "examIds": [
    "upsc-cse"
  ],
  "paperLabel": null,
  "questionImageUrl": null,
  "imageCaption": null,
  "boardDeleted": false,
  "hasAttempted": false,
  "lastAttemptCorrect": null,
  "isBookmarked": false
}
```

## GET /pyq/:id/details (and GET /pyq/:id/reveal)

Both routes build the same object; reveal differs only in being metered and in
returning the explanation to a free user. Shown for a premium user.

### (a) Live APPSC figure question

```json contract=details.figure
{
  "success": true,
  "question": {
    "id": "6f1c2a10-0000-4000-8000-000000000001",
    "text": "Which river is marked as X in the map given below?",
    "optionA": "Krishna",
    "optionB": "Godavari",
    "optionC": "Pennar",
    "optionD": "Vamsadhara",
    "optionE": null,
    "subject": "Geography",
    "topic": "Indian Rivers",
    "year": 2024,
    "difficulty": "MEDIUM",
    "boardDeleted": false,
    "examName": "APPSC Group 1",
    "paperNumber": 1,
    "paperLabel": "PAPER-I",
    "questionImageUrl": "https://example-media-bucket.s3.ap-south-1.amazonaws.com/pyq-figures/3b1e9c2a-5d4f-4e8a-9c1b-7a2d6e0f4c11-appsc-2024-p1-q17.png",
    "imageCaption": "Map of Andhra Pradesh — rivers"
  },
  "lastAttempt": null,
  "correctAnswer": "B",
  "explanation": "The marked river rises near Nashik and enters AP near Bhadrachalam.",
  "approach": null,
  "canViewExplanation": true,
  "unlockedToday": true,
  "attemptNumber": 0,
  "isBookmarked": false,
  "message": "First attempt - good luck!"
}
```

### (b) Board-deleted APPSC question

(`GET /pyq/:id/reveal` on this question answers the 409 above instead.)

```json contract=details.deleted
{
  "success": true,
  "question": {
    "id": "6f1c2a10-0000-4000-8000-000000000002",
    "text": "Consider the following statements about the Polavaram project …",
    "optionA": "1 only",
    "optionB": "2 only",
    "optionC": "Both 1 and 2",
    "optionD": "Neither 1 nor 2",
    "optionE": null,
    "subject": "Economy",
    "topic": "Irrigation",
    "year": 2024,
    "difficulty": "MEDIUM",
    "boardDeleted": true,
    "examName": "APPSC Group 1",
    "paperNumber": 1,
    "paperLabel": "PAPER-I",
    "questionImageUrl": null,
    "imageCaption": null
  },
  "lastAttempt": null,
  "correctAnswer": null,
  "explanation": null,
  "approach": null,
  "canViewExplanation": false,
  "unlockedToday": false,
  "attemptNumber": 0,
  "isBookmarked": false,
  "message": "First attempt - good luck!"
}
```

### (c) UPSC question

```json contract=details.upsc
{
  "success": true,
  "question": {
    "id": "6f1c2a10-0000-4000-8000-000000000003",
    "text": "Which of the following is a Fundamental Duty?",
    "optionA": "To vote",
    "optionB": "To pay taxes",
    "optionC": "To protect the environment",
    "optionD": "To own property",
    "optionE": null,
    "subject": "Polity",
    "topic": "Fundamental Duties",
    "year": 2023,
    "difficulty": "MEDIUM",
    "boardDeleted": false,
    "examName": "UPSC CSE",
    "paperNumber": null,
    "paperLabel": null,
    "questionImageUrl": null,
    "imageCaption": null
  },
  "lastAttempt": null,
  "correctAnswer": "C",
  "explanation": "Article 51A(g).",
  "approach": null,
  "canViewExplanation": true,
  "unlockedToday": true,
  "attemptNumber": 0,
  "isBookmarked": false,
  "message": "First attempt - good luck!"
}
```

## GET /pyq/mock-test — items

Array of these. Board-deleted questions are never served, so every item has a string
`correctAnswer`.

### (a) Live APPSC figure question

```json contract=mock.figure
{
  "id": "6f1c2a10-0000-4000-8000-000000000001",
  "question": "Which river is marked as X in the map given below?",
  "optionA": "Krishna",
  "optionB": "Godavari",
  "optionC": "Pennar",
  "optionD": "Vamsadhara",
  "optionE": null,
  "correctAnswer": "B",
  "answerExplanation": "The marked river rises near Nashik and enters AP near Bhadrachalam.",
  "solveTip": null,
  "subject": "Geography",
  "topic": "Indian Rivers",
  "year": 2024,
  "difficulty": "MEDIUM",
  "paperLabel": "PAPER-I",
  "questionImageUrl": "https://example-media-bucket.s3.ap-south-1.amazonaws.com/pyq-figures/3b1e9c2a-5d4f-4e8a-9c1b-7a2d6e0f4c11-appsc-2024-p1-q17.png",
  "imageCaption": "Map of Andhra Pradesh — rivers"
}
```

### (b) Board-deleted question

Not served — excluded from the pool.

### (c) UPSC question

```json contract=mock.upsc
{
  "id": "6f1c2a10-0000-4000-8000-000000000003",
  "question": "Which of the following is a Fundamental Duty?",
  "optionA": "To vote",
  "optionB": "To pay taxes",
  "optionC": "To protect the environment",
  "optionD": "To own property",
  "optionE": null,
  "correctAnswer": "C",
  "answerExplanation": "Article 51A(g).",
  "solveTip": null,
  "subject": "Polity",
  "topic": "Fundamental Duties",
  "year": 2023,
  "difficulty": "MEDIUM",
  "paperLabel": null,
  "questionImageUrl": null,
  "imageCaption": null
}
```
