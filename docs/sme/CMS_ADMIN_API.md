# CMS Admin API — Unified Content Management Guide

## Overview

This API provides a unified CMS for managing PYQ (Previous Year Questions), Mains questions, and Psychometric content on the UPSC platform. All endpoints live under the `/cms` prefix and are public admin routes — no authentication required. Operations include full CRUD, bulk creation (up to 100 items in an all-or-nothing transaction), and paginated listing with filters.

BASE_URL: https://app.stanzasoft.ai

**Base URL:** `{{BASE_URL}}/api/v1`
**Authentication:** None required (all endpoints are public/admin)
**Note:** All endpoints use the `/api/v1/` global prefix.

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

**Response (201):** Returns the created question with a generated UUID `id`, `createdAt`, and `updatedAt`.

---

### Create a Mains Question

```
POST /cms/mains
Content-Type: application/json
```

**Request Body:**

```json
{
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
| `subject`    | string  | —       | Filter by subject (case-insensitive)           |
| `year`       | integer | —       | Filter by exact exam year                      |
| `difficulty` | enum    | —       | `EASY`, `MEDIUM`, `HARD`, `EXPERT`             |
| `search`     | string  | —       | Search within question text (case-insensitive) |
| `isActive`   | boolean | —       | Filter by active status (`true`/`false`)       |
| `isVerified` | boolean | —       | Filter by verified status (`true`/`false`)     |

**Request Examples:**

```
GET /cms/pyq?page=1&limit=20
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
| `subject`    | string  | —       | Filter by subject (case-insensitive)           |
| `year`       | integer | —       | Filter by exact exam year                      |
| `difficulty` | enum    | —       | `EASY`, `MEDIUM`, `HARD`, `EXPERT`             |
| `search`     | string  | —       | Search within question text (case-insensitive) |
| `isActive`   | boolean | —       | Filter by active status (`true`/`false`)       |
| `isVerified` | boolean | —       | Filter by verified status (`true`/`false`)     |

**Request Examples:**

```
GET /cms/mains?page=1&limit=20
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

**Errors:** `404` — Resource not found, `400` — Validation failed

#### DELETE /cms/mains/:id

Hard delete — permanently removes the question from the database.

**Errors:** `404` — Resource not found

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

### Mains Question Fields

| Field               | Type     | Required | Default | Notes                                    |
| ------------------- | -------- | -------- | ------- | ---------------------------------------- |
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

### Test Set Fields

| Field         | Type     | Required | Default | Notes                                |
| ------------- | -------- | -------- | ------- | ------------------------------------ |
| `name`        | string   | yes      | —       | Display name for the test set        |
| `description` | string   | no       | —       | Brief description of the test set    |
| `questionIds` | string[] | yes      | —       | Array of Psychometric question UUIDs |
| `isActive`    | boolean  | no       | `true`  | Visibility flag                      |

---

## Curl Examples

### PYQ — Create Single

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/pyq \
  -H "Content-Type: application/json" \
  -d '{
    "examName": "UPSC CSE",
    "year": 2024,
    "paperNumber": 1,
    "subject": "History",
    "topic": "Modern India",
    "question": "Who founded the Indian National Congress?",
    "optionA": "Mahatma Gandhi",
    "optionB": "A.O. Hume",
    "optionC": "Jawaharlal Nehru",
    "optionD": "Bal Gangadhar Tilak",
    "correctAnswer": "B",
    "answerExplanation": "Allan Octavian Hume founded the INC in 1885.",
    "difficulty": "EASY"
  }'
```

### PYQ — List with Filters

```bash
curl "{{BASE_URL}}/api/v1/cms/pyq?subject=History&year=2024&difficulty=EASY&page=1&limit=10"
```

### PYQ — Bulk Create

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/pyq/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      {
        "examName": "UPSC CSE",
        "year": 2024,
        "paperNumber": 1,
        "subject": "History",
        "topic": "Ancient India",
        "question": "The Indus Valley Civilization was primarily located in?",
        "optionA": "Ganges Basin",
        "optionB": "Indus-Ghaggar-Hakra river system",
        "optionC": "Deccan Plateau",
        "optionD": "Brahmaputra Valley",
        "correctAnswer": "B",
        "difficulty": "EASY"
      },
      {
        "examName": "UPSC CSE",
        "year": 2024,
        "paperNumber": 1,
        "subject": "History",
        "topic": "Ancient India",
        "question": "Mohenjo-daro is located in present-day?",
        "optionA": "India",
        "optionB": "Pakistan",
        "optionC": "Afghanistan",
        "optionD": "Bangladesh",
        "correctAnswer": "B",
        "difficulty": "EASY"
      }
    ]
  }'
```

### Mains — Create Single

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/mains \
  -H "Content-Type: application/json" \
  -d '{
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
curl "{{BASE_URL}}/api/v1/cms/mains?subject=GS Paper 1&year=2024&page=1&limit=10"
```

### Mains — Bulk Create

```bash
curl -X POST {{BASE_URL}}/api/v1/cms/mains/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      {
        "examName": "UPSC CSE Mains",
        "year": 2024,
        "subject": "GS Paper 2",
        "topic": "Governance",
        "question": "Discuss the role of civil services in policy implementation.",
        "difficulty": "MEDIUM"
      },
      {
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

### Psychometric Question — Create Single

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

1. **Populate questions:** Use bulk create endpoints to upload batches of PYQ, Mains, or Psychometric questions (up to 100 per request)
2. **Review:** List questions with filters to verify data quality — check `isVerified: false` items
3. **Verify:** `PATCH` individual questions to set `isVerified: true` after review
4. **Organize (Psychometric):** Create test sets by grouping verified Psychometric question UUIDs
5. **Deactivate:** `PATCH` with `{ "isActive": false }` to hide content from users without deleting
6. **Clean up:** Use `DELETE` only for permanently removing incorrect or duplicate entries

---

## Common Errors

| Status | Cause                         | Fix                                                                      |
| ------ | ----------------------------- | ------------------------------------------------------------------------ |
| 400    | Missing required field        | Check the Field Reference tables above for required fields               |
| 400    | Unique constraint violation   | A record with the same unique key combination already exists             |
| 400    | Bulk create exceeds 100 items | Split into multiple requests of 100 or fewer                             |
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
