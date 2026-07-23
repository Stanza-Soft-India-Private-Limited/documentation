# Dify latency & cost observability (B7)

**Status:** capture primitive + read API shipped. **Two call sites still need a one-line hook**
(files owned by concurrent work — see [§6](#6-follow-ups--handoff)).

Chat is the core product and we had **zero** latency or cost data on it. `api_usage` times *our*
handler (`/quota/begin`), not the Dify call the user actually waits on, so "the AI is slow" or
"it failed" had nothing behind it.

**No schema change.** Samples are `auth_events` rows with `event_type = 'dify_call'`; the payload
rides in the existing generic `metadata` Json column.

---

## 1. Ground truth — which Dify calls traverse this backend

Established by reading the code, not from memory. **Most of Dify does not touch us at all.**

| Dify call | Path | Where | Instrumentable here? |
|---|---|---|---|
| `POST /messages/:id/feedbacks` — chat thumb forward | **backend → Dify** | `modules/feedback/services/dify-feedback.service.ts` | **Yes** — needs the hook in §6 |
| `/diffy/*` generic pass-through + SSE proxy | **backend → Dify**, but **dead** | `common/http-client/http-client.controller.ts` + `.service.ts` | Yes, but see below |
| Chat turn (Mentor / Expert) | **client → Dify direct** | mobile / web | No — client must emit |
| Mains evaluation workflow | **client → Dify direct** | mobile | No — client must emit |
| PYQ variation, flashcard, mnemonic | **client → Dify direct** | mobile | No — client must emit |

Two corrections to previously-recorded architecture facts:

- **Mains evaluation does *not* run through the backend.** The client runs the Dify workflow itself
  and only `POST`s the finished result to `/mains/:id/evaluation` (`SaveEvaluationDto` =
  `{score, marks, evaluationMarkdown, timeTaken}`). `QuotaController` says so explicitly:
  *"client→Dify direct features (chat, flashcard, mnemonic, mains_eval, pyq_variation)"*.
  `timeTaken` on that DTO is the **user's writing time**, not Dify's — do not read it as latency.
- **The `/diffy/*` proxy is inert.** `HttpClientService` reads `DIFFY_BACKEND_URL`, which is set in
  neither `.env` nor `.env.production`, so it falls back to the literal placeholder
  `https://your-diffy-backend.com`. The configured Dify base URL is `DIFFY_URL`
  (`https://portal.stanzasoft.ai/v1`) and only `DifyFeedbackService` reads it. Instrument the proxy
  if it is ever revived; until then it produces no traffic.

So today **exactly one Dify call traverses this backend**, and the honest consequence is in §3.

---

## 2. What is captured

`DifyMetricsService` (`modules/chat/services/dify-metrics.service.ts`) is the write primitive.
One `auth_events` row per call:

| Column | Value |
|---|---|
| `event_type` | `dify_call` |
| `user_id` | our user id, when attributable |
| `error_code` | short machine code (`ECONNABORTED`, `http_502`, `no_api_key`) |
| `metadata` | flat JSON, below |

`metadata` keys — every one optional except the first three, **omitted when unknown**:

| Key | Meaning |
|---|---|
| `operation` | `message_feedback` · `chat_message` · `workflow_run` · `proxy` (closed set) |
| `outcome` | `success` · `failure` · `timeout` |
| `durationMs` | **our** wall clock around the call |
| `difyApp` | logical app name from `DIFY_APP_KEYS` (`mentor`, `sme_history`) — never the key |
| `status` | HTTP status, when there was one |
| `errorCode` | mirror of the column |
| `promptTokens` `completionTokens` `totalTokens` | only if Dify returned a usage block |
| `totalPrice` `currency` | only if Dify returned a price |
| `difyLatencyMs` | Dify's *self-reported* latency, distinct from `durationMs` |

`timeout` is deliberately split out from `failure`: it is the signature of the **60s proxy idle
timeout** that kills blocking `/workflows/run` calls, and conflating the two is precisely what makes
a slowness report un-diagnosable.

### Never captured

Prompt text, response text, `answer`, conversation content, error *messages* (a code only), and none
of `auth_events`' identity columns. Chat content is the most sensitive data in the product — the
UxCam no-video decision was made to protect exactly this. There is a spec asserting an error message
containing user text does not reach the row.

---

## 3. Cost data: what is genuinely available

**Right now: none — and the API says so rather than showing a zero.**

Dify returns usage in two shapes, and the extractor reads both:

| Dify endpoint | Usage shape |
|---|---|
| `/chat-messages`, `/completion-messages` | `data.metadata.usage = { prompt_tokens, completion_tokens, total_tokens, total_price, currency, latency }` (`latency` in **seconds**) |
| `/workflows/run` (and the `workflow_finished` SSE event) | `data.data = { total_tokens, elapsed_time, … }` — **tokens yes, price no** |
| `/messages/:id/feedbacks` | `{"result":"success"}` — **no usage block at all** |

The only backend-mediated call is the third one. So until either the proxy is revived or the client
starts emitting samples, `tokens.samples` and `cost.samples` will be **0**.

> `cost.samples == 0` means **we do not know**. It does **not** mean the calls were free.
> `cost.available` is returned explicitly so a dashboard cannot render `total: 0` as a cost.

Nothing is estimated or back-filled. There is a spec pinning "no usage block ⇒ no token/cost keys".

---

## 4. Read API

Server-to-server, `x-api-key` (same `@SmeApiKey()` decorator as every other api-key route: `@Public()`
to bypass the global JwtAuthGuard, plus `ApiKeyGuard`). Responses are **raw** — `ResponseInterceptor`
is defined but never registered, so there is no `{success, data}` envelope.

**Base:** `{{BASE_URL}}/api/v1` · **Swagger group:** `SME`
**Data is deploy-forward only** — there is no history before this shipped.

### `GET /chat/observability/dify?days=7`

`days` 1–120, default 7.

```jsonc
{
  "windowDays": 7, "since": "…", "until": "…",
  "totals":  { "calls": 200, "success": 190, "failure": 6, "timeout": 4, "successRatePct": 95 },
  "latencyMs": { "samples": 200, "avg": 820, "p50": 610, "p90": 1800, "p95": 2600, "p99": 8200, "max": 9100 },
  "tokens": { "samples": 0, "total": 0, "prompt": 0, "completion": 0 },
  "cost":   { "samples": 0, "total": 0, "currency": null, "available": false },
  "byOperation": [ { "operation": "message_feedback", "difyApp": "mentor", "calls": 200,
                     "success": 190, "failure": 6, "timeout": 4,
                     "avgMs": 820, "p50Ms": 610, "p95Ms": 2600, "maxMs": 9100,
                     "tokenSamples": 0, "totalTokens": 0, "costSamples": 0, "totalPrice": 0 } ],
  "topErrors":   [ { "errorCode": "ECONNABORTED", "operation": "workflow_run", "count": 4 } ]
}
```

Percentile fields are `null` on an empty window (not `0`).

### `GET /chat/observability/dify/recent?days=7&limit=50`

`limit` 1–200, default 50. Newest-first individual samples for incident triage —
`{ id, at, userId, operation, difyApp, outcome, durationMs, status, errorCode, totalTokens, totalPrice }`.

### ⚠️ Placement

This belongs under `/sme/*` with the rest of the portal analytics. It lives under the chat module
because `src/modules/sme/**` was owned by concurrent work when it was written. Moving it is a
file move + `@Controller` prefix change; the service and the stored rows are unaffected.

---

## 5. Implementation notes

**Non-interference is the load-bearing property.** `DifyMetricsService.record()` returns `void`
synchronously, the Prisma insert is un-awaited and `.catch()`-ed, and the whole body is wrapped so a
malformed sample, a dead DB or a synchronously-throwing client cannot escape. `time()` resolves with
the exact value the call resolved with and **rethrows the original error object unchanged**. There
are specs for each of these, including "the write is still pending when the caller is served".

**Written straight through Prisma, not via `DiagnosticsService`.** That service is the *public
ingest* path: it rate-limits to 60/hr per device and would silently drop trusted server-side writes.
Same reasoning, same precedent as `QuotaService.recordConversationPointer` writing
`chat_conversation_started`.

**Reads use `$queryRaw`** because `metadata` is a `json` (not `jsonb`) column, so Prisma's JSON path
filters do not apply. Every numeric extraction is regex-guarded —
`CASE WHEN metadata ->> 'k' ~ '^-?[0-9]+(\.[0-9]+)?$' THEN (…)::numeric END` — never a bare cast: a
client-emitted event could carry a non-numeric value and one bad row must not 500 the report.
Verified against PostgreSQL 15 with a deliberately poisoned row.

**Provenance is enforced, not assumed.** `dify_call` is on `AUTH_EVENT_TYPES`, which means an
*unauthenticated* caller can post one — deliberately, see 6b — so the row's existence proves nothing
about who wrote it. Without a filter, `curl -d '{"eventType":"dify_call","metadata":{"durationMs":
900000,"totalPrice":50000}}'` would put a fabricated p99 and a $50,000 bill on the SME dashboard
with `cost.available: true`. So every write stamps a **server-controlled** marker in
`metadata.source` (`src/modules/diagnostics/constants/event-source.ts`): `DifyMetricsService` writes
`'backend'`, and the ingest path writes `'client'` **over whatever the body said**, so a caller
cannot claim to be the backend. `DifyObservabilityService` reads
`metadata ->> 'source' = 'backend'` only, and says so in `meta.source` on both responses.
Client-emitted samples are still stored — surfacing them needs a separate, explicitly-labelled read,
not a wider filter. Rows written before the marker shipped have no `source` and are excluded; this
endpoint is deploy-forward only, so there is no history to lose.

**The percentile query is row-capped.** `percentile_cont` is a sorting aggregate — it materialises
every input row — and a 120-day window over a client-emittable event type is unbounded by
construction. `summary()` therefore reads at most `SAMPLE_ROW_CAP` (200,000) rows, newest first, and
reports `meta.sampleRowCap` / `meta.rowCapHit` so a truncated window is visible rather than silent.

**Retention.** `auth_events` has no purge job today, unlike `api_usage` (30d raw / 120d rollup).
At current volume that is fine; if the client starts emitting per-turn samples it will need one.

---

## 6. Follow-ups / handoff

### 6a. Two one-line hooks in files owned by concurrent work

`ChatModule` **exports** `DifyMetricsService`, so each is an import + a wrap.

**`modules/feedback/services/dify-feedback.service.ts`** — the live call. Import `ChatModule` in
`FeedbackModule`, inject `DifyMetricsService`, and wrap the existing post:

```ts
const response = await this.metrics.time(
  { operation: 'message_feedback', difyApp: params.mode === 'sme' ? `sme_${params.subject}` : 'mentor', userId: params.user },
  () => firstValueFrom(this.http.post(/* …unchanged… */)),
);
```

`time()` reads the status off the Axios response, so the service's existing `status >= 200 && < 300`
check keeps working and a non-2xx is already recorded as `failure`. The `no_api_key` early return is
worth a direct `this.metrics.record({ operation: 'message_feedback', outcome: 'failure',
durationMs: 0, errorCode: 'no_api_key' })`.

**`common/http-client/http-client.controller.ts`** — only if `DIFFY_BACKEND_URL` is ever configured.
Note the SSE handlers write to `res` directly, so the honest duration is measured from the
`response.data` `'end'` / `'error'` events, not from when the handler returns.

### 6b. Client-emitted samples (the actual chat latency)

The user-visible chat wait is **client → Dify direct** and this backend cannot see it. To close that
gap the mobile/web client must POST to the existing public sink `POST /api/v1/diagnostics/events`
with the identical payload shape, so the same read queries cover it with no branch:

```jsonc
{
  "eventType": "dify_call",
  "platform": "ios", "appVersion": "1.7.0", "deviceId": "…",
  "metadata": {
    "operation": "chat_message",        // or "workflow_run" for mains_eval / pyq_variation
    "difyApp": "mentor",                // or sme_<subject>
    "outcome": "success",               // success | failure | timeout
    "durationMs": 4820,                 // first byte→completion, measured on device
    "status": 200,
    "totalTokens": 460                  // ONLY if the client actually sees Dify's usage block
  }
}
```

Emit it **after** the turn completes, fire-and-forget, and never block the UI on it. Do not send the
prompt, the answer, or any excerpt of either.

Do **not** send a `metadata.source` key — the ingest overwrites it with `"client"` regardless. Note
the consequence: these rows do **not** appear in `GET /chat/observability/dify`, which reports only
what this backend observed (see "Provenance is enforced" above). Putting client latency on the
dashboard is a follow-up that needs its own clearly-labelled panel, because the two populations are
not comparable — one is a trusted measurement, the other is an unauthenticated assertion.

### 6c. Change outside chat-module ownership — **done**

`src/modules/diagnostics/constants/auth-event-types.ts` now contains `'dify_call'`:

```ts
  'chat_conversation_started',
+ 'dify_call',
```

That allowlist gates the **public ingest endpoint only**; the server-side writes above go straight
through Prisma and never needed it. What the line *does* change is the trust model — the sink is
`@Public()`, so `dify_call` became forgeable the moment it was added, which is exactly why the
provenance marker and the backend-only read filter exist.
