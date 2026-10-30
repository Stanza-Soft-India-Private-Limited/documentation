# SME — Content Language API Guide

> Changed 2026-10-30 — new surface: a `language` on every content row, language maps on the
> JSON copy (paywall, campaigns, filter-config names, surveys, notification rules), and the
> `X-Lang` header on the app and SME list routes. Same day: every example replaced with a
> live staging capture; §5 metrics-vs-list claim and the notification-copy mixed-keys error
> corrected. Later the same day (B23, product ruling): **reels are a shared feed; `language`
> is a tag** — `GET /reels/feed` is no longer language-aware (§0, §5).

Reference for the **SME portal** to enter content in more than one language (Hindi first) and
to read it back. Same server-to-server model as the rest of the SME surface: we expose the
endpoints, the SME team builds the UI.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `BASE_URL` = `https://app.stanzasoft.ai`)
**Authentication:** `x-api-key: <API_KEY_SECRET>` header on **every** `/sme/*` request.
**Swagger:** `/api/docs` (group **SME**).

> **Examples are live.** Every request/response pair below was captured on 2026-10-30 from
> `https://staging-api.prepmonkey.com` running image `f7e0871` (the content-language build),
> redacted (`<API_KEY_SECRET>`, `<app user JWT>`). Ids are real staging ids. Long bodies are
> trimmed where marked (`…`, `…(+N more)`, "other fields omitted"); every field shown is
> verbatim. Rows prefixed `[STAGING TEST hi]` are staging test data. The one exception is the
> campaign example (§3.3), which could not be produced on that image.

> **There is no response envelope.** Success bodies are raw; errors are shaped by the
> exception filter (`success: false`, `message`, `error`, `statusCode`, …). Branch on the HTTP
> status code, never on a `success` field. Match 400s on `message`.

> **Invoke the `frontend-design` skill before building any screen from this doc.** The
> language control is ONE dropdown fed by `GET /sme/i18n/languages` — on every content form
> and every copy editor — not a free-text box.

> **Production prerequisite.** Production does not have `i18n_languages` yet (the UI
> localization rollout, [SME_LOCALIZATION_API.md](./SME_LOCALIZATION_API.md), has to land
> there first). Until then this surface exists on **staging only**.

---

## 0. The mental model (read this first)

**A language is a property of a row, not a translation of one.** A Hindi PYQ is another
`pyq_papers` row with its own id, `language: "hi"`, the same `subject`/`topic` KEYS as the
English rows, and Latin `A`–`E` answer letters. There is no "English twin" and no canonical
id: the Hindi and English folders are filled independently, and they do not have to match
one-for-one.

**Every content table carries `language`** — a code from `GET /sme/i18n/languages`
(`i18n_languages.code`, a foreign key, so it cannot be mistyped into the database):
`pyq_papers`, `mains_papers`, `content_documents` (its quiz questions inherit it), `reel_items`
(its blog inherits it — a TAG only, see "Reels" below), `simulations` (its questions inherit it), `daily_quotes`,
`psychometric_test_sets` (its questions inherit it). Every existing row is `en`. The wire key
stays `language`; its VALUE changed from the dead `"ENGLISH"` to `"en"`.

**Which responses carry `language`** (corrected 2026-10-30 — not literally "every
response"; the key is on every content ROW a response returns, never on aggregates):

| Carries `language` | Where |
|---|---|
| SME rows | every row of `GET /sme/content/pyq`, `/mains`, `/simulations`, `GET /content-doc-admin`, and every create/PATCH response (the full row) |
| App list rows | `GET /pyq`, `GET /mains`, `GET /content-doc`, `GET /reels/feed` (the reel's tag — the feed itself is shared, not language-filtered), `GET /psychometric/test-sets`, `GET /simulations` items, `GET /pyq/mock-test` items |
| App by-id reads | `GET /pyq/:id`, `GET /pyq/:id/details` and `/reveal` (as `question.language`), `GET /mains/:id`, `GET /content-doc/:id` |

`GET /simulations`, `GET /pyq/mock-test`, `GET /pyq/:id/details` and `GET /content-doc/:id`
gained the key on 2026-10-30 (B20-F4); the rest always had the column. **No `language` key**
(not rows): `/pyq/filters`, `/mains/filters`, the `/metrics` routes, `/content-doc/roadmap`
(per-subject summaries — `servedLang` per subject when `X-Lang` was sent),
`/pyq/weak-topics`, `/pyq/wrong-answers` items (projections built from the user's own
attempts), and `/config` (the quote's shape is frozen).

**Copy that lives on a config row is a language MAP** — the exam paywall, campaign content,
filter-config subject display names, survey text, notification-rule copy. English stays
exactly where it is today; other languages are stored beside it. The write API takes today's
plain object (= English) **or** a map `{ "en": …, "hi": … }`; the GET keeps today's field
**and adds** a `…ByLanguage` sibling.

**Subject and topic strings are identifiers.** They drive filters, deep links, icons and
quota keys, so Hindi rows keep the English keys; what a Hindi user reads as the subject name
is the filter-config `displayName` in Hindi (§3.2).

**What the app gets.** The app sends `X-Lang: <its UI language>` (one control: content
language = UI language). Lists are answered per **folder** (= the exact list filter): if the
folder has rows in the requested language it is served in that language, otherwise the same
folder in English — never a mix. By-id reads ignore the language (a shared Hindi row opens
in Hindi for anyone). Details in §5.

**Reels are a shared feed; `language` is a tag** (product ruling 2026-10-30). Reels are NOT
split by language: `GET /reels/feed` returns the same reels, in the same order (newest first),
for every `X-Lang` — Hindi and English reels together — with no fallback, no `servedLang` and
no `X-Served-Lang`. `/reels/bookmarks`, `GET /reels/:id` and the reels-drop notification
count are language-blind too. Set `language` on a reel (`PUT /reels/bulk`, §1.2) to record
what language it is in: every feed row carries it, the reel's blog inherits it (write the
blog in the reel's language), and the coverage script (§6) counts by it — but the tag never
decides who sees the reel. No reel list filters by language
(`GET /reels`, `GET /sme/blog`).

**Allowed languages = any row of `i18n_languages`**, enabled or not, published or not — so
content can be loaded before a language goes live. The app only offers languages that are
enabled **and** published, so unpublished content stays invisible. A **disabled** language
is never served on the app routes even when a client asks for it: `X-Lang: <disabled code>`
is answered in English (`X-Served-Lang: en`); the SME lists still filter by it (§2).

---

## 1. Writing language-tagged rows

### 1.1 `/sme/content/*` — STRICT

`POST /sme/content/pyq`, `POST /sme/content/pyq/bulk`, `PATCH /sme/content/pyq/:id`, the
same three for `mains`, and `POST /sme/content/simulations`, `PATCH /sme/content/simulations/:id`
accept an optional **`language`**:

| Sent | Stored |
|---|---|
| omitted (create) | `en` |
| omitted (PATCH) | unchanged |
| a catalogued code, e.g. `"hi"` (enabled or disabled) | that code |
| a catalogued code in another CASE or with surrounding spaces — `"HI"`, `" hi "`, `"Hi"` | normalised: `hi` (accepted, 200) |
| the legacy names `"ENGLISH"`, `"HINDI"`, `"Hindi"`; a regional code that is not itself catalogued (`"hi-IN"`); `"xx"` | **400**, nothing written |

(Corrected 2026-10-30: an earlier version said `"HI"` is a 400 here — the value is
lower-cased and trimmed before the catalogue check, so it is accepted and stored as `hi`.
Only the legacy NAMES are refused on this surface; the lenient writers in §1.2 and the
language-map keys in §3 do accept them.)

The 400, verbatim:

```
Unknown language "xx". Use a code from GET /sme/i18n/languages (e.g. "en", "hi"), or create it via POST /sme/i18n/languages first.
```

Create a Hindi PYQ — HTTP **201**, the full row comes back with `language: "hi"`:

```http
POST /api/v1/sme/content/pyq
x-api-key: <API_KEY_SECRET>
Content-Type: application/json

{
  "examIds": ["appsc-group-1"],
  "examName": "APPSC Group-I Prelims 2024 Paper-I",
  "year": 2023,
  "subject": "Economy",
  "topic": "Money, Banking and Monetary Policy",
  "difficulty": "EASY",
  "source": "PYQ_THEME",
  "nature": "CORE",
  "question": "[STAGING TEST hi V1b] doc capture: भारत में मौद्रिक नीति समिति (MPC) की अध्यक्षता कौन करता है?",
  "optionA": "वित्त मंत्री",
  "optionB": "RBI गवर्नर",
  "optionC": "नीति आयोग के उपाध्यक्ष",
  "optionD": "मुख्य आर्थिक सलाहकार",
  "correctAnswer": "B",
  "answerExplanation": "RBI अधिनियम, 1934 की धारा 45ZB: RBI गवर्नर MPC के पदेन अध्यक्ष होते हैं।",
  "totalMarks": 1,
  "duration": 120,
  "tags": ["staging-test-hi-v1b", "Economy"],
  "keywords": ["Money, Banking and Monetary Policy"],
  "relatedTopics": ["Money, Banking and Monetary Policy"],
  "language": "hi",
  "paperNumber": 1
}
```

```json
{
  "id": "035af42e-c473-4774-9026-49293da2ec06",
  "examName": "APPSC Group-I Prelims 2024 Paper-I",
  "year": 2023,
  "paperNumber": 1,
  "subject": "Economy",
  "topic": "Money, Banking and Monetary Policy",
  "difficulty": "EASY",
  "source": "PYQ_THEME",
  "nature": "CORE",
  "questionNumber": null,
  "question": "[STAGING TEST hi V1b] doc capture: भारत में मौद्रिक नीति समिति (MPC) की अध्यक्षता कौन करता है?",
  "optionA": "वित्त मंत्री",
  "optionB": "RBI गवर्नर",
  "optionC": "नीति आयोग के उपाध्यक्ष",
  "optionD": "मुख्य आर्थिक सलाहकार",
  "optionE": null,
  "correctAnswer": "B",
  "paperLabel": null,
  "questionImageUrl": null,
  "imageCaption": null,
  "boardDeleted": false,
  "answerExplanation": "RBI अधिनियम, 1934 की धारा 45ZB: RBI गवर्नर MPC के पदेन अध्यक्ष होते हैं।",
  "solveTip": null,
  "prediction": null,
  "weightage": null,
  "repeatFrequency": 0,
  "totalMarks": 1,
  "duration": 120,
  "markingScheme": null,
  "tags": ["staging-test-hi-v1b", "Economy"],
  "keywords": ["Money, Banking and Monetary Policy"],
  "relatedTopics": ["Money, Banking and Monetary Policy"],
  "isActive": true,
  "isVerified": false,
  "language": "hi",
  "createdAt": "2026-10-30T03:52:35.542Z",
  "updatedAt": "2026-10-30T03:52:35.542Z",
  "examIds": ["appsc-group-1"]
}
```

The same body with `"language": "xx"` — HTTP **400**, nothing written:

```json
{
  "success": false,
  "message": "Unknown language \"xx\". Use a code from GET /sme/i18n/languages (e.g. \"en\", \"hi\"), or create it via POST /sme/i18n/languages first.",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-10-30T03:52:35.579Z",
  "path": "/api/v1/sme/content/pyq",
  "method": "POST"
}
```

With the legacy name `"language": "HINDI"` — HTTP **400**, the same message naming the value as
sent: `Unknown language "HINDI". Use a code from GET /sme/i18n/languages …`.

**Bulk is all-or-nothing:** every item's `language` is checked before the first write; one
bad item refuses the whole batch. Every write is audited as `CONTENT_INGEST` with the
language (a bulk write lists every language in the batch).

Simulation questions (`POST /sme/content/simulations/questions/bulk`) carry no language of
their own. **Guidance, not enforced:** link a simulation only to questions in its own
language — create the Hindi questions and assign them into a Hindi simulation. The server
does NOT check this; a mixed link is accepted and the app shows the questions exactly as
linked.

### 1.2 `/cms/*`, `/content-doc-admin`, reels bulk — LENIENT (scripts)

The legacy, script-facing writers accept `language` too, but never 400 on it (every import
script in existence calls them):

| Sent | Stored |
|---|---|
| omitted | `en` (create) / unchanged (update) |
| `"ENGLISH"` / `"HINDI"` (any case) | `en` / `hi` |
| a catalogued code | that code |
| unknown or malformed | `en`, plus a **server WARN** naming the value |

Covers `/cms/pyq` (+ bulk), `/cms/mains` (+ bulk), `/cms/simulations`,
`/cms/psychometric/test-sets` (create/update), `/content-doc-admin` (create/update — the
create used to hard-code `"ENGLISH"`), and `PUT /reels/bulk` (`videos[].language`).
Reels uploaded through the Mux upload routes are `en`; re-tag them with the bulk update. A
reel's `language` is a tag only — it does not move the reel in or out of anyone's feed (§0).

### 1.3 Bulk script

`scripts/pyq-ingest`: `build_rows_v2.py --lang hi` writes `"language": "hi"` on every body
(the default `en` writes no key, so the bodies still post to an older server);
`ingest.py --target staging --lang hi post …` stamps every body and refuses a file that names
another language (without `--lang`, a file that mixes languages is refused).

---

## 2. Listing and filtering by language (SME)

`GET /sme/content/pyq`, `GET /sme/content/mains`, `GET /sme/content/simulations` and
`GET /content-doc-admin` read the **`X-Lang` header** (there is no `?lang=` query parameter —
the global validation would 400 it):

| `X-Lang` | Rows |
|---|---|
| absent | **every language** (the admin view — today's behaviour) |
| `hi` | Hindi rows only |
| a DISABLED language, e.g. `zz` | that language's rows only (content can be prepared before launch — the app routes answer it in English) |
| several values, e.g. `hi, en` | the FIRST value decides (`hi` here) — same rule as the app routes (§5) |
| unknown, e.g. `xx` | **400**, the message from §1.1 |

```http
GET /api/v1/sme/content/pyq?exam=appsc-group-1&limit=5
x-api-key: <API_KEY_SECRET>
X-Lang: hi
```

HTTP 200 — every row is `language: "hi"`; SME lists send no `X-Served-Lang`. First row shown,
the other four trimmed (all `"language": "hi"`); `meta` verbatim (staging's `total` moved while
a seeder was writing Hindi rows):

```json
{
  "data": [
    {
      "id": "8f13ba3d-fd2a-4f52-826a-137ebb537dd1",
      "examName": "APPSC Group-I Prelims 2026 Paper-I",
      "year": 2026,
      "paperNumber": 1,
      "subject": "Economy",
      "topic": "Agriculture and Allied Activities",
      "difficulty": "EASY",
      "source": "PYQ_THEME",
      "nature": "CORE",
      "questionNumber": null,
      "question": "[STAGING TEST hi] भारत में फ़सलों के न्यूनतम समर्थन मूल्य (MSP) की सिफ़ारिश कौन-सी संस्था करती है?",
      "optionA": "नीति आयोग",
      "optionB": "भारतीय खाद्य निगम (FCI)",
      "optionC": "कृषि लागत एवं मूल्य आयोग (CACP)",
      "optionD": "राष्ट्रीय कृषि और ग्रामीण विकास बैंक (NABARD)",
      "optionE": null,
      "correctAnswer": "C",
      "paperLabel": null,
      "questionImageUrl": null,
      "imageCaption": null,
      "boardDeleted": false,
      "answerExplanation": "कृषि लागत एवं मूल्य आयोग (CACP) MSP की सिफ़ारिश करता है; अंतिम निर्णय आर्थिक मामलों की मंत्रिमंडलीय समिति (CCEA) लेती है।",
      "solveTip": null,
      "prediction": null,
      "weightage": null,
      "repeatFrequency": 0,
      "totalMarks": 1,
      "duration": 120,
      "markingScheme": null,
      "tags": ["staging-test-hi", "Economy"],
      "keywords": ["Agriculture and Allied Activities"],
      "relatedTopics": ["Agriculture and Allied Activities"],
      "isActive": true,
      "isVerified": false,
      "language": "hi",
      "createdAt": "2026-10-29T20:26:56.710Z",
      "updatedAt": "2026-10-29T20:26:56.710Z",
      "examIds": ["appsc-group-1"]
    },
    "…(+4 more)"
  ],
  "meta": { "total": 77, "page": 1, "limit": 5, "totalPages": 16, "hasNext": true, "hasPrev": false }
}
```

The same request with `X-Lang: xx` — HTTP **400**, the §1.1 message (`Unknown language "xx". …`,
`"path": "/api/v1/sme/content/pyq?exam=appsc-group-1&limit=5"`, `"method": "GET"`).

Every row carries `language`. `GET /sme/exams/:id/content-counts` stays exam-wide (all
languages together).

---

## 3. JSON copy — language maps

Every editor below follows the same three rules:

1. **Write** today's plain value (= English, exactly as before) **or** a map
   `{ "en": …, "hi": … }`. A map may omit `en` to touch only translations. **Every map key
   must name a catalogued language**: keys are normalised first — case and surrounding
   spaces (`"HI"`, `" hi "` → `hi`) and the legacy names (`"ENGLISH"` → `en`, `"HINDI"` /
   `"Hindi"` → `hi`) — and anything else (`"hi-IN"` when only `hi` exists, `"xx"`,
   `"Hinglish"`) is a **400** with the §1.1 message naming the key as sent; nothing is
   written. Never a silent 200 (corrected 2026-10-30: `{ "HI": {…} }` used to be read as a
   plain English object and saved nothing). Also a 400: two keys for one language
   (`{ "hi": …, "HI": … }`), and — for the object copies (paywall, campaign, notification
   copy) — mixing English fields with language keys in one object
   (`{ "ctaLabel": "BUY", "hi": {…} }`): send `{ "en": { "ctaLabel": "BUY" }, "hi": {…} }`.
   How a map is recognised: for the object copies, a key whose value is an object and that
   is not one of the copy's own fields; for display names / survey text / survey questions,
   any object at all (their plain value is a string or an array).
2. **Merge, never replace**: each language is merged/sanitised against what is stored for
   THAT language; a plain English edit keeps every stored translation.
3. **Read**: the GET keeps today's field (English only, unchanged shape) and adds
   `…ByLanguage` (`{ "en": …, "hi": … }`). The app receives one plain object per request,
   English for any field that is not translated.

### 3.1 Exam paywall copy — `PATCH /sme/exams/:id { paywallContent }`

GET / list / PATCH responses: `paywallContent` (English) + `paywallContentByLanguage`
(`null` when nothing is stored).

```http
GET /api/v1/sme/exams/appsc-group-1
x-api-key: <API_KEY_SECRET>
```

HTTP 200 — the two copy fields (every other exam field omitted, see
[SME_EXAMS_API.md](./SME_EXAMS_API.md)):

```json
{
  "id": "appsc-group-1",
  "paywallContent": {
    "benefits": [
      "All APPSC prelims PYQs with explanations",
      "Unlimited answer reveals",
      "Mentor chat in Telugu-state context"
    ],
    "ctaLabel": "GET APPSC PREMIUM",
    "titleLine1": "Crack APPSC Group 1",
    "titleLine2": "with Premium.",
    "socialProof": "Join APPSC aspirants preparing smarter",
    "benefitsTitle": "Member Benefits",
    "highlightWord": "APPSC",
    "highlightStyle": "CORAL",
    "extraComparisonRows": []
  },
  "paywallContentByLanguage": {
    "en": {
      "benefits": [
        "All APPSC prelims PYQs with explanations",
        "Unlimited answer reveals",
        "Mentor chat in Telugu-state context"
      ],
      "ctaLabel": "GET APPSC PREMIUM",
      "titleLine1": "Crack APPSC Group 1",
      "titleLine2": "with Premium.",
      "socialProof": "Join APPSC aspirants preparing smarter",
      "benefitsTitle": "Member Benefits",
      "highlightWord": "APPSC",
      "highlightStyle": "CORAL",
      "extraComparisonRows": []
    },
    "hi": {
      "benefits": [
        "सभी APPSC Prelims PYQ, व्याख्या सहित",
        "असीमित उत्तर देखने की सुविधा",
        "तेलुगु राज्यों के संदर्भ में मेंटर चैट"
      ],
      "ctaLabel": "APPSC Premium लें",
      "titleLine1": "APPSC Group 1 में सफलता पाएँ",
      "titleLine2": "Premium के साथ।",
      "socialProof": "APPSC के अभ्यर्थियों के साथ समझदारी से तैयारी करें",
      "benefitsTitle": "सदस्यता के लाभ",
      "highlightWord": "APPSC"
    }
  }
}
```

A map PATCH that touches only Hindi (staging was restored right after):

```http
PATCH /api/v1/sme/exams/appsc-group-1
x-api-key: <API_KEY_SECRET>
Content-Type: application/json

{
  "paywallContent": {
    "hi": {
      "titleLine1": "APPSC जीतें",
      "benefits": ["सभी APPSC Prelims PYQ, व्याख्या सहित"],
      "ctaLabel": "प्रीमियम लें"
    }
  }
}
```

HTTP 200 — `paywallContent` (English) is unchanged; `paywallContentByLanguage.hi` took the
three sent fields and **kept** the four it did not send (`titleLine2`, `socialProof`,
`benefitsTitle`, `highlightWord`); `en` is unchanged — identical to the GET above, omitted
here with every other exam field:

```json
{
  "paywallContentByLanguage": {
    "hi": {
      "benefits": ["सभी APPSC Prelims PYQ, व्याख्या सहित"],
      "ctaLabel": "प्रीमियम लें",
      "titleLine1": "APPSC जीतें",
      "titleLine2": "Premium के साथ।",
      "socialProof": "APPSC के अभ्यर्थियों के साथ समझदारी से तैयारी करें",
      "benefitsTitle": "सदस्यता के लाभ",
      "highlightWord": "APPSC"
    }
  }
}
```

A key in another case is normalised — `{ "paywallContent": { "HI": { "ctaLabel": "APPSC Premium लें" } } }`
→ HTTP 200 and the value lands under `paywallContentByLanguage.hi` (there is no `HI` key in
the response). The three §3 rule 1 refusals, each HTTP **400** with nothing written:

| Sent `paywallContent` | `message` |
|---|---|
| `{ "hi-IN": { "ctaLabel": "x" } }` | `Unknown language "hi-IN". Use a code from GET /sme/i18n/languages (e.g. "en", "hi"), or create it via POST /sme/i18n/languages first.` |
| `{ "ctaLabel": "BUY", "hi": { "ctaLabel": "x" } }` | `paywallContent mixes language keys (hi) with fields (ctaLabel). Send either the plain (English) value, or a map of language code → value, e.g. { "en": {…}, "hi": {…} }.` |
| `{ "hi": { "ctaLabel": "a" }, "HI": { "ctaLabel": "b" } }` | `Language keys "hi" and "HI" both mean "hi" — send each language once.` |

Translatable: `titleLine1`, `titleLine2`, `highlightWord`, `benefitsTitle`, `benefits`,
`socialProof`, `ctaLabel`, `extraComparisonRows`. Never per language: `highlightStyle`, plans,
prices. ⚠️ An exam with no English paywall copy keeps the hand-written English comparison
table even when a translation exists — only the copy lines are translated there.

A stored empty list (`"benefits": []`, `"extraComparisonRows": []` — "none", a decision) is
kept by every PATCH that does not set that field, in English and in each translation;
`null` clears it. (Corrected 2026-10-30: any PATCH used to drop a stored `[]`.)

### 3.2 Subject display names — `PUT /sme/filter-config/:examType/:subject { displayName }`

`displayName` takes a string (= English), `null` (clear English), or
`{ "en": "Polity", "hi": "राजव्यवस्था" }` (`null` for a language clears that language). The
subject KEY never changes. GET `effective` gains `displayNameByLanguage`; the app's
`/pyq/filters` and `/mains/filters` return `subjectsMeta[].displayName` in the requested
language, sorted on it. See [SME_FILTER_CONFIG_API.md](./SME_FILTER_CONFIG_API.md).

```http
GET /api/v1/sme/filter-config/prelims?exam=appsc-group-1
x-api-key: <API_KEY_SECRET>
```

HTTP 200 — top-level keys `examType, exam, validIconKeys, entries`; the Economy entry (a Hindi
name stored in `override.displayNameI18n`, English from the default):

```json
{
  "subject": "Economy",
  "isOptional": false,
  "questionCount": 208,
  "effective": {
    "displayName": "Economy",
    "iconKey": "sme_economy",
    "hidden": false,
    "sortOrder": null,
    "displayNameByLanguage": {
      "hi": "अर्थव्यवस्था",
      "en": "Economy"
    }
  },
  "override": {
    "displayNameI18n": {
      "hi": "अर्थव्यवस्था"
    }
  }
}
```

### 3.3 Campaign content — `POST /sme/offers`, `PATCH /sme/offers/:id { content }`

`content` is today's object (= English) or `{ "en": {…}, "hi": {…} }`; each language goes
through the same sanitiser (`styles`, avatars). Translatable: `badgeText`, `titleLine1`,
`titleLine2`, `highlightWord`, `urgencyTitle`, `countdownLabel`, `benefitsTitle`, `benefits`,
`comparisonTitle`, `comparison`, `socialProof`, `ctaLabel`. Never per language: `styles`,
`socialProofAvatars`, `plans` (titles/prices stay as authored). GET: `content` (English) +
`contentByLanguage`.

> **Example: not producible on staging until B21 (`e148635`) is deployed** — campaign writes
> failed with `400 {"message": "Database error", "error": "DatabaseError"}` on image
> `f7e0871`, for every create that passed validation, including one with plain English
> content (not language-related) — see commit `e148635`. No `contentByLanguage` body could be
> captured, so none is shown here. What the capture did prove: the language checks run before
> the write and answer as documented — `{ "xx": {…} }` and `{ "hi-IN": {…} }` → the §1.1 400
> (create and PATCH), `{ "ctaLabel": …, "hi": {…} }` →
> `content mixes language keys (hi) with fields (ctaLabel). …`, `{ "hi": …, "HINDI": … }` → the
> duplicate-key 400.

### 3.4 Surveys — `POST /sme/surveys`, `PATCH /sme/surveys/:id`

`title` / `description`: a string or `{ "en": "…", "hi": "…" }`. `questions`: the array
(= English, gets the ids) or `{ "en": [...], "hi": [...] }`. **A translation aligns BY
POSITION**: same number of questions, same `type` at each position, exactly the English
option `key`s — only `text` and option `label`s are translated; anything else is a 400 that
names the position. Answers always use the English ids/keys, so one survey has one result
set whatever language people answered in. Re-sending English questions re-keys stored
translations when the shape is unchanged; if it changed, the 400 asks for every translation
in the same request. Questions (any language) are frozen once the survey is ACTIVE; titles
are not. GET: today's fields + `titleByLanguage`, `descriptionByLanguage`,
`questionsByLanguage` (full arrays in English order — re-sendable).

```http
GET /api/v1/sme/surveys/3571ea0e-3dd7-4a9a-825a-a31d8a68f910
x-api-key: <API_KEY_SECRET>
```

HTTP 200 (verbatim) — note the Hindi questions carry the **English** ids and option `key`s:

```json
{
  "id": "3571ea0e-3dd7-4a9a-825a-a31d8a68f910",
  "title": "[STAGING TEST hi] Content language check",
  "description": "Two quick questions about studying in Hindi. Staging test survey, never activated.",
  "status": "DRAFT",
  "questions": [
    {
      "id": "d32da4e5-f9fc-4097-9344-cdd7775533d3",
      "text": "Which language do you prefer for study material?",
      "type": "single_choice",
      "options": [
        { "key": "en", "label": "English" },
        { "key": "hi", "label": "Hindi" },
        { "key": "both", "label": "Both" }
      ]
    },
    {
      "id": "3e9d21a5-f3e8-4bd2-9298-b356c88f5db0",
      "text": "How clear was the Hindi content you read today?",
      "type": "rating_1_5"
    }
  ],
  "targetTiers": [],
  "targetPlatforms": [],
  "targetAspirantTypes": [],
  "targetExams": [],
  "priority": 1,
  "startAt": null,
  "endAt": null,
  "activatedAt": null,
  "closedAt": null,
  "createdAt": "2026-10-29T20:34:37.893Z",
  "updatedAt": "2026-10-30T03:47:30.227Z",
  "titleByLanguage": {
    "en": "[STAGING TEST hi] Content language check",
    "hi": "[STAGING TEST hi] सामग्री की भाषा पर आपकी राय"
  },
  "descriptionByLanguage": {
    "en": "Two quick questions about studying in Hindi. Staging test survey, never activated.",
    "hi": "हिंदी में पढ़ाई के बारे में दो छोटे प्रश्न। स्टेजिंग परीक्षण सर्वे — कभी सक्रिय नहीं किया जाएगा।"
  },
  "questionsByLanguage": {
    "en": [
      {
        "id": "d32da4e5-f9fc-4097-9344-cdd7775533d3",
        "text": "Which language do you prefer for study material?",
        "type": "single_choice",
        "options": [
          { "key": "en", "label": "English" },
          { "key": "hi", "label": "Hindi" },
          { "key": "both", "label": "Both" }
        ]
      },
      {
        "id": "3e9d21a5-f3e8-4bd2-9298-b356c88f5db0",
        "text": "How clear was the Hindi content you read today?",
        "type": "rating_1_5"
      }
    ],
    "hi": [
      {
        "id": "d32da4e5-f9fc-4097-9344-cdd7775533d3",
        "text": "अध्ययन सामग्री के लिए आप कौन-सी भाषा पसंद करते हैं?",
        "type": "single_choice",
        "options": [
          { "key": "en", "label": "अंग्रेज़ी" },
          { "key": "hi", "label": "हिंदी" },
          { "key": "both", "label": "दोनों" }
        ]
      },
      {
        "id": "3e9d21a5-f3e8-4bd2-9298-b356c88f5db0",
        "text": "आज आपने जो हिंदी सामग्री पढ़ी, वह कितनी स्पष्ट थी?",
        "type": "rating_1_5"
      }
    ]
  },
  "counts": { "answered": 0, "snoozed": 0, "dismissed": 0 }
}
```

### 3.5 Notification rules — `POST /sme/notification-rules`, `PATCH /sme/notification-rules/:id { copy }`

`copy` is `{ "title", "body" }` (= English) or `{ "hi": { "title", "body" } }`.
`{ "title": "", "body": "" }` removes a translation. A translation must be complete and use
**exactly** the English `{{placeholders}}` (a missing or extra one is a 400 naming it). Each
recipient gets the copy in their **app UI language** (`user_profiles.ui_language`), English
otherwise; a test send shows it. `enable` still only requires English. GET already returns
the whole map, keyed by language — English is stored under `en`.

```http
GET /api/v1/sme/notification-rules/7675e008-510b-45cd-8cea-634b6833cc8a
x-api-key: <API_KEY_SECRET>
```

HTTP 200 (verbatim; `unavailableReason` cut at `…`):

```json
{
  "id": "7675e008-510b-45cd-8cea-634b6833cc8a",
  "triggerKey": "pib_today",
  "triggerKind": "SWEPT",
  "name": "Pib Today",
  "examId": null,
  "category": "CONTENT",
  "channel": "PUSH_FEED",
  "isEnabled": false,
  "rolloutPercent": 0,
  "isTransactional": false,
  "config": {},
  "copy": {
    "en": {
      "body": "The current-affairs points that matter, in one place.",
      "title": "Today's PIB digest is ready"
    },
    "hi": {
      "body": "ज़रूरी करेंट अफ़ेयर्स बिंदु, एक ही जगह पर।",
      "title": "आज का PIB सार तैयार है"
    }
  },
  "deepLink": { "type": "library" },
  "audience": {},
  "sendAtIst": "07:30",
  "cooldownDays": 1,
  "priority": 50,
  "isLiveNow": false,
  "available": false,
  "unavailableReason": "PIB content does not exist in the product yet — no model, table or publish date. Implement resolve() once the content source is decided. Bac…",
  "createdAt": "2026-08-26T02:25:03.906Z",
  "updatedAt": "2026-10-29T20:49:58.253Z"
}
```

⚠️ Mixing English fields with a language key in `copy` (`{ "title": …, "body": …, "hi": {…} }`)
is a **400**, but it is refused by request validation before the language check, so the
`message` is the validation ARRAY `["copy property hi should not exist"]`, not the
"mixes language keys" sentence the paywall and campaign copy return. Match on the status.

---

## 4. Hand-edited config (no SME writer)

- **Exam-phase copy** — `app_config.filter_config.examPhases.copyI18n`:
  `{ "hi": { "revision": { "label": "…", "headline": "…", "focus": "…" } } }`, field by field
  over code defaults (Hindi defaults ship in code, **pending SME review**) and the English.
- **Daily quotes** — `daily_quotes.language`; `/config` picks today's quote from the
  requested language's own active pool, falling back to the English pool. Add Hindi quotes
  with SQL — there is no write route. ⚠️ `daily_quotes.id` is `text NOT NULL` with **no
  database default** (the uuid is generated by the app's ORM, not by Postgres), so a hand
  `INSERT` must supply it:

  ```sql
  INSERT INTO daily_quotes (id, text, language)
  VALUES (gen_random_uuid()::text, '…', 'hi');
  ```
- **Daily-task titles** — seeded tasks are shown translated at serve time from a code map
  (Hindi **pending SME review**); a user's own tasks are never touched.

---

## 5. The public contract — what the apps call

- **`X-Lang: <code>`** on app requests (read case-insensitively). Absent, blank, unknown,
  malformed or **disabled** → **English, 200** (never a 400 on app routes; when the header
  was sent, `X-Served-Lang: en` / `servedLang: "en"` say so). `hi-IN` → `hi` when only `hi`
  exists. The legacy names are honoured here too: `X-Lang: HINDI` (any case) → `hi`,
  `X-Lang: ENGLISH` → `en`. **More than one value** — two `X-Lang` headers (which arrive
  joined as `"hi, en"`) or a comma-separated list in one header — → the **first** non-blank
  value (`hi` here). Send one header with one code.
- **Folder-level fallback** on `/pyq`, `/mains`, `/content-doc`, `/content-doc/roadmap` (per
  subject), `/simulations`, `/psychometric/test-sets`, `/pyq/mock-test`. The
  `/metrics` routes probe with the list's own filter, so they pick the same language as the
  list, and for a language-resolved folder the counts match the list (APPSC Hindi Economy:
  list 14, `totalQuestions` 14). (Corrected 2026-10-30: this used to say counts *always*
  match.) ⚠️ **Known gap on the English side** — pre-existing, not language-related (the old
  image showed it too): on APPSC the English folders list more rows than `/metrics` counts —
  Economy 196 vs 194, Polity 210 vs 206, all subjects 2068 vs 2051 (staging, 2026-10-30;
  APPSC was the exam compared). Do not assert list total == `totalQuestions` for an English
  folder.
  `/pyq/filters` and `/mains/filters` list the union of the requested language and English,
  so every chip opens a non-empty folder.
- **Language-blind by design:** every by-id read, `?bookmarked=true`, `/reels/bookmarks`,
  `/pyq/wrong-answers`, `/pyq/weak-topics`, `/content-doc/continue-reading` — and
  **`/reels/feed`** (2026-10-30 ruling: reels are a shared feed, `language` is a tag — the
  same body for every `X-Lang`, no `servedLang` / `X-Served-Lang`, no `Vary: X-Lang`).
- **Which language was served:** the `X-Served-Lang` response header, and on object
  responses a `servedLang` body field — **both only when the request sent `X-Lang`**, so a
  client that sends no header gets byte-identical responses.
- **Copy routes** (`/paywall`, `/config`, `/tasks/today`, `/feedback/surveys/*`) return the
  same shapes with the copy in the requested language, English per field where untranslated.
- **Cache:** `/pyq/filters`, `/mains/filters` vary by `X-Lang` (`Vary: X-Lang`; the cache key
  only changes when the header is sent). `/config`, `/paywall` and `/tasks/today` are not
  server-cached but their bodies depend on the header, so they send `Vary: X-Lang` too.
- ⚠️ **`/pyq/filters` and `/mains/filters` are cached for 1 hour** (per exam × language).
  After creating content — especially the FIRST Hindi rows of a subject, or a new subject —
  the new chip / `displayName` can take up to an hour to appear there, while `/pyq` and
  `/mains` themselves list the rows at once. Check with the list, not with the filters.
- Observed, not a promise: other language-dependent responses — the `/pyq`, `/mains` and
  `/content-doc` lists, the `/metrics` routes, `/feedback/surveys/current` — do **not** send
  `Vary: X-Lang` (staging, 2026-10-30). A client-side cache must key on `X-Lang` itself.

### 5.1 Examples (app routes, `Authorization: Bearer <app user JWT>`, `X-Exam: appsc-group-1`)

**Filters are the union.** `GET /api/v1/pyq/filters` with `X-Lang: hi` — HTTP 200,
`x-served-lang: hi`, `vary: Origin, X-Lang, Accept-Encoding` (arrays trimmed):

```json
{
  "subjects": ["Art & Culture", "Current Affairs", "Economy", "…(+8 more)"],
  "years": [2026, 2025, 2024, "…(+11 more)"],
  "difficulties": ["EASY", "MEDIUM", "HARD"],
  "subjectsMeta": [
    { "subject": "Art & Culture", "displayName": "Art & Culture", "iconKey": "sme_art", "isOptional": false },
    { "subject": "Current Affairs", "displayName": "Current Affairs", "iconKey": "sme_current_affairs", "isOptional": false },
    { "subject": "Environment & Ecology", "displayName": "Environment & Ecology", "iconKey": "sme_environment", "isOptional": false },
    "…(+8 more)"
  ],
  "papers": ["MARCH", "OCTOBER", "PAPER-I", "…(+4 more)"],
  "servedLang": "hi"
}
```

The same call without `X-Lang` — HTTP 200, no `x-served-lang`, no `servedLang`, still
`vary: Origin, X-Lang, Accept-Encoding`:

```json
{
  "subjects": ["Art & Culture", "Current Affairs", "Economy", "…(+8 more)"],
  "years": [2024, 2022, 2019, "…(+8 more)"],
  "difficulties": ["EASY", "MEDIUM", "HARD"],
  "subjectsMeta": [
    { "subject": "Art & Culture", "displayName": "Art & Culture", "iconKey": "sme_art", "isOptional": false },
    { "subject": "Current Affairs", "displayName": "Current Affairs", "iconKey": "sme_current_affairs", "isOptional": false },
    { "subject": "Economy", "displayName": "Economy", "iconKey": "sme_economy", "isOptional": false },
    "…(+8 more)"
  ],
  "papers": ["MARCH", "OCTOBER", "PAPER-I", "…(+4 more)"]
}
```

Compared in full (untrimmed): the `subjects` keys are identical; `years` with `hi` adds
`2026, 2025, 2023` (years that only Hindi rows have) and drops none; with `hi` two
`displayName`s are Hindi — `Economy → अर्थव्यवस्था`, `Polity → राजव्यवस्था` — and
`subjectsMeta` is sorted on the displayed name, so those two sort last; without the header no
name is translated.

**By-id is language-blind.** The Hindi row from §1.1, opened with `X-Lang: en`:

```http
GET /api/v1/pyq/035af42e-c473-4774-9026-49293da2ec06
Authorization: Bearer <app user JWT>
X-Exam: appsc-group-1
X-Lang: en
```

HTTP 200 — the same row as the §1.1 create response, in Hindi (`"language": "hi"`), with no
`x-served-lang` header and no `servedLang` field; `vary: Origin, Accept-Encoding`.

**`servedLang` only when asked.** `GET /api/v1/pyq/metrics?subject=Economy`:

| `X-Lang` | `x-served-lang` | Body |
|---|---|---|
| `hi` | `hi` | `{ "totalQuestions": 14, "attempted": 2, "unsolved": 12, "servedLang": "hi" }` |
| `zz` (catalogued, **disabled**) | `en` | `{ "totalQuestions": 194, "attempted": 0, "unsolved": 194, "servedLang": "en" }` |
| absent | — | `{ "totalQuestions": 194, "attempted": 0, "unsolved": 194 }` |

(The list for the same folder: `hi` 14 rows; `zz` and absent 196 rows — the English-side gap
noted above.)

---

## 6. Coverage check

```
npx ts-node -T scripts/i18n-content-coverage.ts            # staging
npx ts-node -T scripts/i18n-content-coverage.ts --prod     # production, read only
```
Active rows per exam × table × language, and per language "has / falls back to en"
(`reel_items` is counted but left out of that summary — reels are a shared feed and their
`language` is a tag, §0). It says
"migration … NOT applied" on a database without the `language` columns.

---

## Common errors

| Status | When | `message` |
|---|---|---|
| 400 | `language` (or a map key, or `X-Lang` on an SME list) is not in `i18n_languages` | `Unknown language "xx". Use a code from GET /sme/i18n/languages …` |
| 400 | `"ENGLISH"` / `"HINDI"` as the `language` FIELD on `/sme/content/*` | same — the legacy names are accepted by the lenient writers (§1.2) and as language-map keys (§3), not here (`"HI"` / `" hi "` are fine: case and spaces are normalised) |
| 400 | a language map mixes English fields with language keys — paywall, campaign | `paywallContent mixes language keys (hi) with fields (ctaLabel). Send either the plain (English) value, or a map of language code → value, …` (`content` likewise) |
| 400 | the same, on notification-rule `copy` (`{ "title", "body", "hi": {…} }`) | the validation array `["copy property hi should not exist"]` — request validation refuses it first (corrected 2026-10-30: this table used to say `copy` gets the "mixes language keys" sentence) |
| 400 | two map keys name the same language | `Language keys "hi" and "HI" both mean "hi" — send each language once.` |
| 400 | survey translation does not line up | `questions.hi[1] has type "…" but the English question at that position is "…"` (and the count / option-key variants) |
| 400 | notification translation placeholders differ | `copy.hi must use exactly the English placeholders …; missing {{streak}}` |
| 400 | a translation with a title but no body (rules) | `copy.hi needs both a title and a body (send both empty to remove it).` |
