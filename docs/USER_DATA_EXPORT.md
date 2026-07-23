# User Data Export — `GET /user/me/export`

Self-service export of everything PrepMonkey holds about the **authenticated
caller**. This is the access half of the data-subject rights pair; the erasure
half (`DELETE /user/me` → `UserService.deleteAccount`) already exists and also
satisfies Apple's in-app account-deletion requirement.

| | |
|---|---|
| Route | `GET /user/me/export` |
| Auth | Required (global `JwtAuthGuard`; the route is **not** `@Public()`) |
| Subject | Always the JWT's user. **No id parameter of any kind** |
| Query params | `format` — `json` (default) or `file` |
| Response | Raw JSON. No response envelope (see below) |
| Rate limit | 1 per 120s **and** 3 per IST calendar day, per user → `429` |
| Implementation | `src/modules/user/user-data-export.service.ts` |
| Tests | `src/modules/user/user-data-export.service.spec.ts` |

---

## 1. Why there is no user-id parameter

The endpoint reads the subject from `@CurrentUser()` (the verified JWT) and
passes it to `UserDataExportService.exportForUser(userId)`. There is no path
param, no query param and no SME/admin sibling route.

This is the single most important property of the feature. An export endpoint
that accepts an id is a full-database exfiltration primitive: one leaked or
guessed id returns another person's phone number, payment history, IP addresses
and study record in one request. The DTO
(`src/modules/user/dto/export-user-data.dto.ts`) declares only `format`, and the
global `ValidationPipe` runs with `forbidNonWhitelisted: true`, so any attempt
to smuggle a `userId` query param is a `400` before the handler runs.

Every query in the service is filtered on that one id. `simulation_answers` —
the only exported table without a `userId` column of its own — is reached via
`where: { attempt: { userId } }`, i.e. still fenced by the caller. No `include`
walks upward from a row to a parent that could belong to somebody else. Two
tests enforce this: one asserts that *every* recorded Prisma `where` contains the
caller's id and no other, and one asserts that exporting a different id never
produces a query mentioning the first.

---

## 2. Response shape

```jsonc
{
  "export": {
    "schemaVersion": 1,
    "userId": "…",
    "generatedAt": "2026-07-23T12:34:56.789Z",   // UTC instant
    "generatedAtIST": "2026-07-23T18:04:56+05:30", // IST wall clock
    "filename": "prepmonkey-data-export-2026-07-23.json",
    "truncated": false,
    "truncatedSections": [],
    "notes": [ "…plain-language disclosures, including every known gap…" ]
  },
  "sections": {
    "account": { … },              // single object
    "accountLinks": { … },         // single object
    "profile": { … } | null,       // single object
    "studyGraph": { … } | null,    // single object (Neo4j User node)
    "questionAttempts": {          // every other section is a collection:
      "items": [ … ],
      "returned": 5000,
      "cap": 5000,
      "truncated": true,
      "totalAvailable": 18342      // present ONLY when truncated
    }
    // …
  }
}
```

**The per-section `{ items, returned, cap, truncated }` wrapper is not a global
response envelope.** This repo deliberately has none — `ResponseInterceptor`
exists but is never registered, and the only `APP_INTERCEPTOR` is
`ApiUsageInterceptor`, so success bodies are raw. The wrapper is part of the
export *payload*: without it, a client receiving exactly 5000 rows cannot tell
whether that is the whole record or a silent truncation. Errors keep the usual
`AllExceptionsFilter` shape (`{ success: false, message, error, statusCode, … }`).

Dates are ISO-8601 UTC, with the IST wall-clock also stated on the envelope. The
export date used in the filename and the rate-limit bucket comes from
`getISTDateString()` in `src/common/utils/ist-date.util.ts` — the server runs
UTC but users are in India, so "today" must resolve in IST.

### Sections

| Section | Source table | Cap | Notes |
|---|---|---:|---|
| `account` | `user_auth` | – | Explicit column allowlist |
| `accountLinks` | `user_auth` | – | `provider` + `googleLinked`/`appleLinked` booleans |
| `profile` | `user_profiles` | – | Whole row (no excluded columns) |
| `studyGraph` | Neo4j `User` node | – | Streak/hours. Best-effort, `null` on graph outage |
| `psychometricResults` | `psychometric_test_results` | 200 | Includes the user's own `answers` JSON |
| `questionAttempts` | `user_question_attempts` | 5000 | Own selection + ids; not the question bank |
| `documentProgress` | `user_document_progress` | 2000 | Joined to document title/subject/topic only |
| `simulationAttempts` | `simulation_attempts` | 500 | Joined to simulation name |
| `simulationAnswers` | `simulation_answers` | 5000 | Flat, scoped via parent attempt |
| `customTasks` | `custom_tasks` | 5000 | Whole row |
| `userContent` | `user_content` | 1000 | Flashcards/mnemonics/revisions incl. HTML |
| `orders` | `orders` (+ nested `payments`) | 500 | |
| `appleSubscriptions` | `apple_subscriptions` | 100 | |
| `refunds` | `refunds` | 200 | |
| `disputes` | `disputes` | 200 | |
| `notificationHistory` | `notification_history` | 2000 | Personalized sends only |
| `deviceTokens` | `device_tokens` | 100 | Token **masked** to last 6 |
| `sessions` | `sessions` | 200 | Token hashes excluded |
| `feedbackReports` | `feedback_reports` (+ `replies`) | 500 | |
| `surveyResponses` | `survey_responses` | 500 | Includes the user's own `answers` JSON |
| `chatMessageFeedback` | `chat_message_feedback` | 1000 | |
| `authEvents` | `auth_events` | 500 | Login/OTP diagnostics incl. IP + device |
| `apiUsage` | `api_usage` | 500 | Raw request log, ~30-day retention |

---

## 3. Format

**JSON, one document.** The data spans 20 tables with nested rows and JSON
columns; CSV would need roughly twenty files and would flatten away the
structure that makes the export legible.

`?format=file` returns the identical body but adds
`Content-Type: application/json; charset=utf-8` and
`Content-Disposition: attachment; filename="prepmonkey-data-export-<IST date>.json"`,
so a browser saves it. Those download mechanics follow the repo's existing
file-download precedent, `SmeSurveyService.exportCsv`; only the media type
differs. The controller uses `@Res({ passthrough: true })` so Nest still
serializes the returned object — the body stays raw JSON and nothing bypasses the
normal response path.

---

## 4. Size and truncation

Nothing is fetched unbounded.

* Every collection has a per-section cap (table above), sized so the worst-case
  document stays in the low single-digit MB after the globally-enabled gzip
  compression.
* Each section is fetched with `take: cap + 1`. If the probe row comes back, the
  section is marked `truncated: true`, the extra row is dropped, and **only then**
  a `count()` runs so the response can state the true `totalAvailable`. An
  ordinary user therefore pays exactly one query per section.
* `export.truncated` and `export.truncatedSections` summarise it at the top, and
  a plain-language sentence naming the affected sections is appended to
  `export.notes`. A truncated export never claims to be complete.
* Truncation always keeps the **most recent** rows (ordered by an indexed column
  wherever the schema has one).
* Sections are fetched **sequentially, not with `Promise.all`**. This endpoint
  shares the production database; ~20 concurrent queries from a single request
  would occupy a large slice of the Prisma connection pool. Sequential execution
  keeps the footprint at one pooled connection, and the rate limit makes the
  extra latency irrelevant.
* `simulation_answers` is exported as its own flat capped section rather than
  nested under attempts, so a user with many attempts cannot multiply the cap
  (500 attempts × ~100 answers would be 50 000 rows).

Streaming was considered and rejected: with caps in place the payload is small
enough to buffer, and a streamed body cannot carry the `truncated` contract the
client needs.

**If a user hits a cap and wants the remainder**, the honest current answer is a
manual request to support. Pagination was deliberately not added: a paginated
compliance export invites callers to walk the whole table on a shared production
database, and the caps are set well above realistic usage.

---

## 5. Rate limiting

Uses the existing `RateLimitService` (`src/common/services/rate.limiting.service.ts`),
following the `DiagnosticsService` pattern.

| Window | Key | Limit |
|---|---|---|
| Burst, 120s | `user:export:burst:<userId>` | 1 |
| Daily, 86 400s | `user:export:<userId>:<IST date>` | 3 |

The burst window is checked **first**, deliberately: `checkRateLimit` increments
on every call, so checking the daily bucket first would let a client hammering
the endpoint burn its whole daily allowance on requests that were about to be
rejected anyway. Checking burst first means abusive traffic consumes only the
cheap short-window counter and the scarce daily quota survives for legitimate
use.

Two differences from the diagnostics precedent, both intentional:

* keyed on the authenticated `userId` (diagnostics is a pre-auth sink, so it
  keys on deviceId/IP);
* a tripped limit **throws `429`** rather than silently dropping. A user
  exercising a legal right must never be handed a success response containing
  nothing. The body is `{ success: false, message, error: "ExportRateLimited",
  retryAfterSeconds, statusCode: 429, … }`.

Both limits are checked **before any database work**, so a rejected request costs
one Valkey `INCR`.

> Operational note: `CacheService` swallows Redis errors. If Valkey is down,
> `checkRateLimit` will not throttle and the endpoint runs unlimited. That is the
> same failure mode every other rate limit in this codebase has, and it is the
> right trade-off here (fail-open on an access right rather than deny it), but it
> is worth knowing during a Valkey incident.

---

## 6. What is deliberately excluded, and why

Every read of a table that contains anything sensitive uses an explicit Prisma
`select` — an **allowlist, never a denylist**. With `select`, a column added to
the schema tomorrow is not exported until somebody consciously adds it here; with
an `omit`/spread approach, the next token hash somebody adds leaks silently.
Under-exporting is a fixable complaint. Over-exporting a secret is a breach.

### A. Credentials and capabilities

| Excluded | Why |
|---|---|
| `sessions.accessTokenHash`, `sessions.refreshTokenHash` | Bearer-token material. A leak here is an account takeover. |
| `device_tokens.token` | An FCM/APNs push token is the *capability* to push to that device. Exported as `tokenMasked` (last 6 chars) + platform, which is enough to tell devices apart. |
| `payments.razorpaySignature` | HMAC over the payment; verification secret material with zero user value. |
| `user_auth.cognitoSub`, `.googleId`, `.appleId` | Identity-provider subject identifiers, i.e. auth-system lookup keys. Reported instead as `provider` + `googleLinked`/`appleLinked` booleans. |

There is **no password hash to exclude**: authentication is delegated to Cognito
and `user_auth` has no password column at all.

### B. Raw provider payloads and schemaless JSON blobs

| Excluded | Why |
|---|---|
| `payment_events` (whole table, incl. `rawBody`, `signature`) | Raw Razorpay/Apple webhook bodies and their signatures. The user-facing facts they encode are already exported as `orders`, `payments`, `refunds`, `disputes`. |
| `apple_subscriptions.raw`, `refunds.raw`, `disputes.raw` | Signed provider payloads — JWS/receipt material plus provider-internal fields. |
| `orders.notes`, `payments.notes` | Schemaless internal metadata blobs. A `Json` column has no reviewable shape, so a future writer could put anything in it; excluded on principle rather than on trust in today's contents. |
| `orders.lastReconciledAt`, `apple_subscriptions.lastNotificationUUID` | Internal job/webhook bookkeeping. |

**Contrast:** `psychometric_test_results.answers`, `survey_responses.answers` and
`auth_events.metadata` **are** exported. The first two are literally the user's
own answers; the third is bounded and coerced at ingest by `DiagnosticsService`
and never contains free-form data.

### C. Internal-only fields and staff commentary

| Excluded | Why |
|---|---|
| `feedback_report_replies.actorNote` | Internal SME note attached to a reply, never shown to the user. The reply `text` **is** exported. |
| `sme_audit_log` (whole table) | Internal admin action trail: staff commentary (`actorNote`) plus before/after snapshots of internal fields. |
| `user_auth.wyltoSyncHash`, `.wyltoSyncedAt` | CRM sync dedup bookkeeping, meaningless to the user. |
| `chat_message_feedback.difyForwarded`, `.difyError` | Vendor-forwarding status. The feedback the user actually wrote **is** exported. |
| `auth_events.identifierHash` | SHA-256 correlator over their own phone/email. The readable `identifierMasked` is exported instead. |

### D. Other people's data, and things that are not personal data

| Excluded | Why |
|---|---|
| Anything not keyed to the caller | No section runs without a `userId == caller` predicate. |
| The content catalogue — `content_documents`, `content_questions`, PYQ and simulation question banks, correct answers, explanations | Licensed product content, not personal data. The user's **own** selected option and the document/question ids they answered **are** exported, so their record stays complete and re-linkable. |
| `topic_notifications` | Broadcasts stored once globally and identical for every user, with no per-user row. |
| `admin_users`, `admin_sessions`, `admin_roles`, `password_resets`, `webhook_events` | Staff and provider-global tables that never reference an app user. |

### E. Known gaps — stated in the payload's own `notes`, not hidden

| Gap | Why | Fix if required |
|---|---|---|
| `api_usage_daily` (≤120-day rollup) | Its only usable index is `@@unique([day, userId, route])`, which is **day-leading**, so a per-user lookup would sequential-scan a shared production table. The ≤30-day **raw** `api_usage` rows it is derived from **are** exported. | Add a `userId`-leading index (a schema change, out of scope here). |
| Neo4j beyond the `User` node | PYQ/Mains attempt and bookmark edges live in the graph with no bounded per-user read today. The `User` node itself (streak, hours) **is** exported, best-effort. | Add bounded per-user graph reads to `UserRepository`. |
| Ephemeral Redis/Valkey state (OTPs, daily quota counters) | TTL'd operational state, not a record of the user. | n/a |

Because these gaps are named in `export.notes` inside every response, the export
never presents itself as more complete than it is.

---

## 7. DPDP position — what this does and does not do

**Read this before describing the feature to anyone as "DPDP compliant".**

**What this endpoint provides:** the *technical mechanism* by which a Data
Principal can obtain the personal data the Data Fiduciary holds about them —
self-service, authenticated, immediate, and machine-readable. Together with
`DELETE /user/me` (erasure), it covers the two rights that are pure engineering
problems.

**What this endpoint does not do — none of this is satisfied by code in this
repo:**

* **It is not a compliance determination.** Nobody has had a lawyer confirm that
  the section list, the exclusions or the caps meet the statutory standard. The
  design principle used here is engineering-conservative ("never leak a secret"),
  which is not the same as legally sufficient.
* **Notice and consent obligations sit entirely elsewhere** — the privacy notice,
  the itemised notice at collection, the consent record and its withdrawal path,
  purpose limitation, and the consent-manager provisions. None of those are
  implemented by this endpoint.
* **There is no verified request workflow.** DPDP contemplates requests from a
  Data Principal *and* from a nominated person or legal heir. This endpoint only
  serves a live, logged-in account holder. A user who has lost account access, or
  a nominee, still needs a manual process.
* **No grievance-redressal mechanism, response SLA, or Data Protection Officer /
  contact designation** is implemented here.
* **Retention and deletion policy is not addressed.** Deleting an account
  cascades most tables, but `payment_events`, `auth_events`, `api_usage` and
  `sme_audit_log` deliberately have no FK and survive deletion by design. Whether
  those retention periods are lawful is a policy question, not a code question.
* **Third-party processors are out of scope.** Data also sits with Wylto (CRM),
  Dify (AI chat), Firebase, Meta, UxCam, Razorpay and Apple. This export covers
  what *our* database holds, not what those processors hold.
* **Known gaps remain** (section 6E), and truncation is possible for very heavy
  users (section 4).

Accurate one-line summary: *"Users can export their own data on demand; this
provides the data-access mechanism, and the policy, notice, consent and
grievance obligations are tracked separately."*

---

## 8. Testing

`src/modules/user/user-data-export.service.spec.ts` mocks Prisma with a proxy
that **honours the `select` argument the way Prisma actually would**, and plants
`SECRET_*` sentinel values in every excluded column of every mocked row. That
makes the exclusion tests prove something about the production query rather than
about the mock. Covered:

* every expected section is present, with the truncation contract on each;
* the catch-all sweep: no `SECRET_` string survives anywhere in the serialized
  export;
* per-field assertions for each entry on the exclusion list (section 6);
* the query itself never *selects* an excluded column, including nested selects
  (`orders.payments`, `feedbackReports.replies`);
* the excluded tables (`payment_events`, `sme_audit_log`, `api_usage_daily`, the
  admin tables, …) are never queried at all;
* every query is scoped to the caller, and exporting a different id never
  mentions the first;
* `simulation_answers` is scoped via `{ attempt: { userId } }`;
* every collection query carries a bounded `take`;
* truncation is reported honestly, `count()` runs only when a section overflowed,
  and exactly-at-cap is not truncation;
* the burst limit is checked before the daily one, both throw `429`, and neither
  performs any database work;
* the push token is masked rather than dropped;
* the Neo4j section degrades to `null` (plus a note) instead of failing the
  export.

`src/modules/user/user.controller.spec.ts` covers the controller: the subject
always comes from the JWT (even if a rogue `userId` reaches the handler),
download headers are set only for `format=file`, and the body is returned
unchanged via passthrough.
