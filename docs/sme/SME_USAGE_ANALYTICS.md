# SME — Usage Analytics & DAU API

> Changed 2026-09-21 — stale-text pass (telemetry live in 2.0, version ladder, links). Exam-dimension changes follow in [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md).
> Changed 2026-09-21 — ?exam= on every analytics route, examCoverage, purchase-funnel (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

Dedicated read-only endpoints for the SME portal to render **Daily Active Users** and
**per-user / per-feature engagement**. Server-to-server, same model as the rest of the SME
surface.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `https://app.stanzasoft.ai`)
**Auth:** `x-api-key: <API_KEY_SECRET>` on every request · **Swagger:** `/api/docs` (group **SME**)
**All endpoints** accept `?days=30` (default 30, max 120) and optional `?startDate=&endDate=` (ISO),
plus an optional **`?exam=<slug>`** — see §0b, and read it before you draw a per-exam chart.
All day-bucketing is **IST** (UTC+5:30). Counts are integers; rates are percentages.

---

## 0. Two data planes (read this first)

| | **Plane A — app tables (live)** | **Plane B — `api_usage` capture** |
|---|---|---|
| Source | the app's own tables (simulation, MCQ, psychometric, doc-reading, AI-content, planner, signups, devices, notifications) | a new per-request capture table written by a global interceptor |
| History | **full, immediately** (read live each call; never purged) | **deploy-day forward only**; raw kept 30d, rollup kept 120d, then purged |
| Powers | engaged-DAU, per-feature usage, signups, retention, premium-vs-free, content, platforms, notifications | true app-wide DAU, per-endpoint usage, heatmap, paywall (402) hits |

**DAU is shown as two labeled series:** `engagedDau` (Plane A — a user who did a *persisted action*; full history, a conservative floor) and `activeDau` (Plane B — any authenticated request; deploy-forward, `null` for pre-deploy days).

**Not captured (intentional, v1):** chat / mains-eval / pyq-variation run **client → Dify directly** (our server never sees them), so they don't appear in `/features`. Capturing them later is a separate Dify decision.

---

## 0b. Exam scoping — `?exam=` (read this second)

Every route in this doc — §1–§4, the three insight routes in §4b and the funnel in §4c —
accepts an optional **`?exam=<slug>`** (slugs come from `GET /sme/exams`; the default exam is
`upsc-cse`). Four rules, then the per-route table.

**1. It is purely additive.** Omit it and the response is **byte-identical** to what the route
has always returned — no new keys, no wrapper, no extra aggregate. An existing portal build
keeps working untouched. An **unknown slug is a `400`**, never a silently empty chart: on a
management screen "0 users" and "you typo'd the slug" look identical and only one is worth
acting on.

**2. "Exam" is not one column, and the routes do not all use the same one.** Picking the wrong
axis answers a different question, so each route names its source:

| Source | Column | What NULL means | History |
|---|---|---|---|
| **PERSON** | `user_profiles.active_exam_id` | **NULL ≡ `upsc-cse`** — the person onboarded before app 2.0 (2026-09-17, which made the exam picker onboarding step 1) and has never used the home switcher, so the default is what they were actually served. Still the majority of the base; shrinks with 2.0 adoption. | **Full**, exact |
| **REQUEST** | `api_usage.exam_id` / `api_usage_daily.exam_id` | **NULL = "not recorded"** → the row is **EXCLUDED**, `upsc-cse` included. The column only exists from the **2026-09-21** deploy. | Deploy-forward; carries `examCoverage` |
| **CONTENT** | `content_documents.exam_ids` / `simulations."examIds"` | — (array overlap; the `*` sentinel matches **every** exam) | Full |
| **ENTITLEMENT** | a live row in `user_exam_entitlements` for that exam | — | Full |

⚠️ The PERSON and REQUEST rules point in **opposite directions on purpose**. NULL
`active_exam_id` genuinely *is* the default exam; NULL `api_usage.exam_id` is genuinely
*unknown*. Do not "harmonise" them.

**3. `examCoverage` — what a REQUEST-sourced per-exam number is allowed to claim.**

<!-- captured from staging 2026-09-21, backend f6329e6 -->
Four real `examCoverage` blocks, all captured within the same minute on 2026-09-21
against `exam=appsc-group-1`. **They disagree, and every one of them is correct:**

```json
// GET /sme/analytics/summary?days=3&exam=appsc-group-1   (examCoverage is TOP-LEVEL here)
{ "engagedDau24h": 0, "engagedWau7d": 3, "engagedMau30d": 3,
  "activeDau24h": 0, "activeMau30d": 0, "stickiness": 0,
  "examCoverage": { "since": "2026-09-21T14:30:34.917Z", "unknownRows": 7473 } }

// GET /sme/analytics/heatmap?days=3&exam=appsc-group-1   (WRAPPED)
{ "data": [], "examCoverage": { "since": "2026-09-21T14:30:34.917Z", "unknownRows": 64 } }

// GET /sme/analytics/dau?days=3&exam=appsc-group-1       (WRAPPED)
{ "data": [], "examCoverage": { "since": null, "unknownRows": 0 } }

// GET /sme/analytics/endpoints?days=3&exam=appsc-group-1 (WRAPPED)
{ "data": [], "examCoverage": { "since": null, "unknownRows": 0 } }
```

> **Why `summary` and `heatmap` report a `since` while `dau` and `endpoints` report
> `null` for the same window and the same exam:** the first two read **raw
> `api_usage`**, which was already being exam-stamped minutes after the deploy; the
> latter two read the **`api_usage_daily` rollup**, which the nightly job had not yet
> written any exam-stamped row into. **`since: null` with `unknownRows: 0` means "this
> source has nothing to say yet"** — it does *not* mean "no traffic" and it does *not*
> mean "no rows were discarded". Render it as *not recorded yet*.
>
> **And `unknownRows` differs between two routes reading the same table** (7473 vs 64)
> because each counts the discards under **its own** predicate and window. It is this
> response's discard count, never a table-wide backlog. Do not sum them, and do not
> show one route's `unknownRows` next to another route's numbers.

| Field | Meaning |
|---|---|
| `since` | Earliest **exam-stamped** row inside the window, or `null` if the whole window predates the column. An **ISO instant** from raw `api_usage`; an **IST `YYYY-MM-DD`** on `dau` / `endpoints`, whose source is the `api_usage_daily` rollup (its grain is a date). |
| `unknownRows` | How many rows in this exact window were **dropped** for carrying no exam. Counted over the same predicate the route itself used, so it is this response's own discard count — not some other slice of the table. |

**What the portal must do with it:** render a *"since &lt;date&gt;"* label on the panel, and
**never compare a per-exam number against a pre-deploy all-exam number**. Per-exam numbers on
these routes start on this release's deploy day. The history is **unknown**, not UPSC — the
correct picture is a start date, not a cliff in usage.

**4. Retro-attribution (PERSON-sourced routes).** Wherever `active_exam_id` is the source, a
user — and everything they ever did — is attributed to the exam they have picked **today**.
There is no switcher history. A signup from June who switched to APPSC last week counts as an
APPSC signup in June. For `upsc-cse` this is almost always a no-op (it absorbs every NULL);
for a newer exam it is a small, one-directional overstatement that grows as people switch.

### Exam source per route

| Route | Exam source | Shape with `?exam=` |
|---|---|---|
| `summary` | **MIXED** — `engaged*` via PERSON, `active*` via `api_usage.exam_id` | same object **+ top-level `examCoverage`** (not wrapped) |
| `dau` | **MIXED**, same split | **wrapped** `{ data, examCoverage }` — `since` is an IST date |
| `features` | PERSON | unchanged shape |
| `users` | PERSON (narrows both the page and `total`) | unchanged shape |
| `signups` | PERSON | unchanged shape |
| `retention` | PERSON (narrows the **cohort**; the activity test is unchanged) | unchanged shape |
| `churn-risk` | PERSON (narrows both the page and `total`) | unchanged shape |
| `premium-engagement` | population PERSON, **tier = live ENTITLEMENT in that exam** | unchanged shape, **different meaning** — see §3 |
| `paywall-hits` | REQUEST (`api_usage.exam_id`) | **wrapped** `{ data, examCoverage }` |
| `heatmap` | REQUEST (`api_usage.exam_id`) | **wrapped** `{ data, examCoverage }` |
| `endpoints` | REQUEST (`api_usage_daily.exam_id`) | **wrapped** `{ data, examCoverage }` — `since` is an IST date |
| `platforms` | PERSON — the token **owner's** exam, not the device's | unchanged shape |
| `content` | **CONTENT** `examIds` (`*` matches all) | unchanged shape |
| `notifications` | PERSON — the **recipient's** exam | unchanged shape |
| `release-health` (§4b) | REQUEST, **except `clientErrors`** | **top-level `examCoverage`** + `meta.examFilter` + `meta.clientErrorsExamScoped: false` |
| `notification-effectiveness` (§4b) | PERSON — the **recipient's** exam | `meta.examFilter` (no `examCoverage`) |
| `onboarding-funnel` (§4b) | PERSON (the signup cohort), **auth_events block app-wide** | **+ `examSelected`** sibling key + `meta.examFilter` |
| `purchase-funnel` (§4c) | PERSON for the five event steps, **`orders.exam_id`** for `ordersPaid` | `exam` echoed + an extra `caveats[]` entry |

⚠️ **Four routes change SHAPE** under `?exam=` — `dau`, `paywall-hits`, `heatmap`, `endpoints`
go from a bare array to `{ data, examCoverage }`. A bare array has nowhere to hang the coverage
note, and shipping the note is the point. The client must branch on
`Array.isArray(res) ? res : res.data`.

⚠️ **Per-exam numbers do not always sum to the all-exams number.** `content` matches the
content's tags, so one `*`-tagged document counts for every exam and the per-exam figures sum
**above** the total. REQUEST-sourced routes exclude unknown-exam rows, so they sum **below** it.

---

## 1. Core

### `GET /sme/analytics/summary`
`{ engagedDau24h, engagedWau7d, engagedMau30d, activeDau24h, activeMau30d, stickiness }` — `stickiness = engagedDau24h / engagedMau30d`.
`?exam=` — **MIXED source**: the `engaged*` trio splits on the PERSON (full history), the `active*` pair on `api_usage.exam_id` (deploy-forward). The object gains a **top-level `examCoverage`** (not a wrapper), computed over the widest window it reports on (MAU = 30d). So `engagedMau30d` and `activeMau30d` are **not** comparable per-exam until `examCoverage.since` is 30 days old.

### `GET /sme/analytics/dau?days=30`
`[{ day, engagedDau, activeDau }]` — `activeDau` is `null` for days before the capture was deployed.
`?exam=` — MIXED, same split as `summary`, and the array is **wrapped**: `{ data: [...], examCoverage }`. `examCoverage.since` is an **IST `YYYY-MM-DD`** here (rollup grain). The per-exam `activeDau` line legitimately starts later than the per-exam `engagedDau` line.

### `GET /sme/analytics/features?days=30`
`[{ feature, users, actions }]` — features: `simulation, mcq, doc_reading, psychometric, ai_flashcard, ai_mnemonic, ai_revision, planner`. (`planner` counts only genuine user-created tasks; auto-seeded rows are excluded.)
`?exam=` — PERSON. Despite reading the same six domain tables the DAU series does, this one has **full per-exam history**: no `api_usage`, so no `examCoverage` and no deploy-day floor.

### `GET /sme/analytics/users?days=30&page=1&limit=20&search=&sort=lastSeen`
Paginated `{ data, total, page, limit, hasMore }`; rows `{ id, email, name, phoneNumber, premiumExpiresAt, lastSeen, actions30d, isPremium }`. `sort ∈ lastSeen|actions`; `search` matches email/name.
`?exam=` — PERSON; narrows **both the page and `total`**. ⚠️ `isPremium` on these rows is still the person-level `premiumExpiresAt > now` mirror — it is **not** re-scoped to the exam. Use `GET /sme/users?premium=&exam=` when you need per-exam entitlement truth in a list.

---

## 2. Growth & Retention

### `GET /sme/analytics/signups?days=30` → `[{ day, newUsers }]`
`?exam=` — PERSON. ⚠️ **Retro-attributed**: a signup is counted under the exam the user has picked *today*, not the one they saw on signup day (§0b rule 4).
### `GET /sme/analytics/retention?days=30`
`{ cohortSize, d1, d7, d30, d1Pct, d7Pct, d30Pct }` — of users who signed up in the window, how many were active ≥1/7/30 days after signup.
`?exam=` — PERSON; narrows the **cohort** only. The activity test is unchanged: a user's actions all belong to them whatever exam they are in.
### `GET /sme/analytics/churn-risk?inactiveDays=14&page=1&limit=20`
Paginated; users active within 90 days but **silent for ≥`inactiveDays`**. Rows `{ id, email, name, phoneNumber, lastActivity }`.
`?exam=` — PERSON; narrows both the page and `total`.

---

## 3. Monetization × Engagement

### `GET /sme/analytics/premium-engagement?days=30`
`[{ tier, users, activeUsers, totalActions, avgActionsPerActiveUser }]` for `tier ∈ premium|free` (premium = `premiumExpiresAt > now`).

⚠️ **`?exam=` changes what "premium" MEANS on this route.** The shape is identical; the
definition is not.

| | **without `?exam=`** | **with `?exam=`** |
|---|---|---|
| Population | every `user_auth` row | the exam's PEOPLE (`active_exam_id`, NULL ≡ `upsc-cse`) |
| `tier: "premium"` | the **person-level mirror** `user_auth.premium_expires_at > now` | a **LIVE ENTITLEMENT ROW** in *that exam* (`user_exam_entitlements`, the shared `liveEntitlementWhere`) |

The mirror flips to subscribed on **any** purchase, so without the switch an exam-scoped call
would report every APPSC buyer as a premium UPSC user. The consequence, stated so it is not
later filed as a bug: **someone who paid for APPSC but has since switched their active exam to
UPSC appears in the UPSC population as `free`.** They *are* free in UPSC — that is what
per-exam entitlements mean. It is the same switch `GET /sme/users?premium=true&exam=` makes.

### `GET /sme/analytics/paywall-hits?days=30`
`[{ route, hits, users }]` — HTTP **402** hits = upgrade-intent. ⚠️ Sourced from raw `api_usage` → limited to the **~30-day** raw window.
`?exam=` — REQUEST (`api_usage.exam_id`): the exam the *request* was made in, which is the right axis for a refusal — a 402 is about the content someone was refused, not about who they are. **Wrapped** as `{ data, examCoverage }`; `since` is an ISO instant.

---

## 4. Behavioural

### `GET /sme/analytics/heatmap?days=30` → `[{ weekday, hour, count }]` (IST; ~30d raw window)
`?exam=` — REQUEST (`api_usage.exam_id`). **Wrapped** as `{ data, examCoverage }`; `since` is an ISO instant.
### `GET /sme/analytics/endpoints?days=30` → `[{ route, calls, users }]` (top 100, from rollup)
`?exam=` — REQUEST (`api_usage_daily.exam_id`). **Wrapped** as `{ data, examCoverage }`; `since` is an **IST `YYYY-MM-DD`** (the rollup's grain is a date).
### `GET /sme/analytics/platforms` → `{ ios, android, web }` (active devices)
`?exam=` — PERSON, matched on the **token owner's** active exam. ⚠️ A device has no exam of its own and one handset can belong to a user who switches, so this reads as *"how the people studying X reach us"*, **not** an install count for X.
### `GET /sme/analytics/content?days=30` → `{ topDocuments:[{title,subject,readers}], simulation:{started,submitted,completionRate} }`
`?exam=` — **CONTENT**, not the reader: `content_documents.exam_ids` / `simulations."examIds"`, overlap-matched so the `*` sentinel counts for **every** exam. A UPSC user reading an APPSC-tagged document is APPSC consumption. ⚠️ Therefore the per-exam numbers can sum **above** the all-exams number, because one shared (`*`) document belongs to every exam.
### `GET /sme/analytics/notifications?days=30` → `{ sent, read, readRate }`
`?exam=` — PERSON, matched on the **recipient's** active exam; `notification_history` carries no exam of its own. Prefer `notification-effectiveness` (§4b) for any decision.

---

## 4b. Release, messaging & acquisition health (see [SME_INSIGHTS_API.md](./SME_INSIGHTS_API.md))

Three further `/sme/analytics/*` endpoints live in their own doc because their caveats
are load-bearing:

### `GET /sme/analytics/release-health?days=14&platform=`
Per **app version × platform**: `activeUsers, requests, errors4xx, errors5xx, errorRatePct,
errorRateExcludingPaywallPct, clientErrors, clientErrorsPerActiveUser, adoptionSharePct`
plus a `comparison` against the next-older build (`verdict ∈ better|worse|similar|insufficient_data`)
and a per-platform rollup with `preHeaderSharePct`. Supplies the number behind an
`AppConfig.minAppVersion` force-update decision.
⚠️ `app_version IS NULL` **is** the pre-1.7 cohort (the header shipped with 1.7) and is
reported as its own labelled bucket — never dropped. Current store build is **2.0** (Android 27,
iOS 2.0 build 3, released 2026-09-17); `AppConfig.minAppVersion` on production is **1.0.0**, so
no forced update is in effect. ⚠️ `app_version`/`platform`/`status`
exist only on **raw** `api_usage`, so the window is hard-capped at **30 days**.

`?exam=` — REQUEST (`api_usage.exam_id`). Adds a **top-level `examCoverage`** (beside
`versions` / `platforms`, not inside `meta`), `meta.examFilter`, and
**`meta.clientErrorsExamScoped: false`** plus two extra `meta.caveats` entries.
⚠️ **`clientErrors` cannot be exam-scoped at all** — it comes from `auth_events`, which has no
exam column and no relation to join one through. So with an exam filter the client-error column
stays **app-wide** while `activeUsers` / `requests` beside it are narrowed, which makes
`clientErrorsPerActiveUser` an **overstatement**, not a comparable per-exam rate. Judge builds
on `errorRateExcludingPaywallPct` when filtering by exam.

### `GET /sme/analytics/notification-effectiveness?days=30`
Per notification `type` and per IST day: `sent, read, maturedSent, maturedRead,
maturedReadRatePct, pending, wastedSends, recipients`, plus an explicit
`worstPerformers` kill-list ranked by wasted sends.
⚠️ **`isRead` is a bulk "opened the in-app feed" flag** set by `markFeedSeen`
(`POST /notifications/feed/seen`) — **not** a per-push open, and push delivery is not
tracked at all. ⚠️ **Time-to-read is derivable only from 2026-07-23 onward.** A `read_at` column was
added that day and `markFeedSeen` now stamps it, but rows marked read BEFORE that have
`read_at = NULL`. Never read NULL as "instant" — it means "read before we recorded when". Rates are quoted on a **matured** sample only (default: rows ≥3 days old).

`?exam=` — PERSON, matched on the **recipient's** `active_exam_id` (NULL ≡ `upsc-cse`). Adds
`meta.examFilter` and one caveat; **no `examCoverage`** (nothing here is `api_usage`-sourced).
⚠️ Read it as *"pushes that landed on people who study X"*, **not** *"pushes about X"* — the
rule engine does not record which exam a send was reasoned about, and a user who switched exam
takes their whole send history with them.

### `GET /sme/analytics/onboarding-funnel?days=30`
Signup → phone verified → onboarding completed → psychometric resolved → first genuine action,
for a bounded **signup cohort**: `{ cohort, summary, stages[], byDay[], authEvents, meta }`
— plus the pre-auth failure signal (`login_failed` / `otp_send_failed` / `otp_verify_failed`)
from `auth_events` over the same window. Full field list and caveats in
[SME_INSIGHTS_API.md §4](./SME_INSIGHTS_API.md).
⚠️ `auth_events` capture began **2026-07-23** and emitter coverage is **split**:
`otp_send_failed` is also emitted server-side (complete), while `login_attempt` /
`login_success` / `login_failed` / `otp_verify_failed` are **mobile-only** and began flowing
with app **2.0** (2026-09-17) — a **floor** whose height is 2.0's *adopted share*, not a census.
Read `authEvents.interpretation` before quoting any failure number.

`?exam=` — PERSON; narrows the **signup cohort**, and adds an **`examSelected`** block plus
`meta.examFilter`. `examSelected` is a **sibling key of `stages[]`, deliberately NOT a sixth
stage** — `stages` is indexed positionally by the portal, and renumbering it for a caller who
never asked about exams would be a silent break. It carries `cohortSignups` (the filtered
number) beside `windowSignups`, `explicitlySelected(+Pct)`, `nullCount` / `nullBucketedAs`, a
`byExam[]` distribution over the **whole** window (NULL already folded into `upsc-cse`), and a
`note`. ⚠️ The **`authEvents` block stays app-wide** — a pre-auth event has no user to join an
exam through — so it sits un-narrowed beside an exam-scoped funnel. Say so on the panel.

> Note: the older `GET /sme/analytics/notifications` (§4 above) returns the flat
> `{ sent, read, readRate }` with **no maturation split and no isRead caveat**. Prefer
> `notification-effectiveness` for any decision.

---

## 4c. `GET /sme/analytics/purchase-funnel?exam=&days=30`

**New 2026-09-21.** Per IST day: `paywallViewed → upgradeTapped → checkoutOpened →
checkoutAbandoned / purchaseFailed → ordersPaid`, with a conversion percentage. It is the only
read that puts the **client's** view of a purchase (`auth_events`) next to the **server's**
(`orders`) — and those two halves have different trust and different exam attribution, which
is the whole reason it is its own endpoint rather than another metric on `/summary`.
The narrative — why it exists, how to visualise it, the three ways it gets misread — is
[SME_INSIGHTS_API.md §5](./SME_INSIGHTS_API.md); this section is the field-level contract.

| Param | Default | Notes |
|---|---|---|
| `days` | `30` | Clamped to **1..90** — `auth_events` is purged at 90 days, so a longer window would lose its own tail. |
| `exam` | — | Optional slug; unknown → `400`. |

**Steps.** The five funnel steps are `count(DISTINCT user_id)` per IST day over
`auth_events.event_type ∈ paywall_viewed, upgrade_tapped, checkout_opened, checkout_abandoned,
purchase_failed`. **Distinct users, not event rows** — a paywall re-renders and a checkout sheet
gets reopened; counting rows would make `paywallViewed` a function of how chatty the build is,
and the conversion rate would fall every time the UI got more responsive.
`ordersPaid` is **server truth** from `orders` (`status = PAID`, `is_test = false` — iOS sandbox
purchases hit production and are excluded from every money surface), bucketed by the order's own
IST creation day.

**`conversionPct` = `ordersPaid ÷ paywallViewed × 100`, one decimal.** Two rules that will
otherwise be read as bugs:
* **`null`, never `0`, on a day with no recorded paywall views.** "No denominator" and
  "converted nobody" are different facts, and the first usually means the day predates the
  emitting build. Branch on `paywallViewed`, not on the rate.
* **It can exceed 100%, and that is information.** The steps are client-emitted and the orders
  are not, so someone who bought without a recorded paywall view is counted in the numerator and
  not the denominator. A `conversionPct > 100` measures **emitter coverage**, not a broken
  paywall. For the same reason a *rising* conversion can mean 2.0 adoption rather than a better
  paywall.

**Exam attribution is asymmetric.** `auth_events` rows carry **no exam of their own**, so with
`?exam=` an event is attributed to the exam the user has picked **at query time**
(`active_exam_id`, NULL ≡ `upsc-cse`) — not the exam whose paywall they were looking at.
`ordersPaid` does **not** share this weakness: `orders.exam_id` records the exam the money
actually bought. So a per-exam `conversionPct` mixes a **retro-attributed denominator** with an
**exactly-attributed numerator** — read it as directional, and read the all-exams number (omit
`exam`) as the exact one. The service ships that sentence verbatim as the last entry of
`caveats` whenever `exam` is supplied.

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/analytics/purchase-funnel?exam=appsc-group-1&days=3` — verbatim, complete
except that the seven `caveats` strings are shown as `"…"` (they are reproduced in full
just below):

```json
{
  "exam": "appsc-group-1",
  "days": 3,
  "series": [
    { "day": "2026-09-21", "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
      "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 0, "conversionPct": null },
    { "day": "2026-09-20", "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
      "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 0, "conversionPct": null },
    { "day": "2026-09-19", "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
      "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 0, "conversionPct": null }
  ],
  "totals": {
    "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
    "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 0, "conversionPct": null
  },
  "caveats": ["…", "…", "…", "…", "…", "…", "…"]
}
```

> **What that capture proves, which a hand-written example cannot:** every day inside the
> window is **present and zero-filled** (three days requested, three returned, newest
> first), and a zero day's `conversionPct` is **`null`, never `0`** — including in
> `totals`. Staging has no paywall traffic, so this is the empty-state body the portal
> will see on any quiet exam-day slice, and it must render as *"no data"*, not as
> *"0% conversion"*.
>
> `caveats` came back with **7 entries** (6 base + the exam one). The full text of all
> seven, verbatim from the same response:
>
> 1. *"The five funnel steps are CLIENT-emitted (paywall_viewed / upgrade_tapped / checkout_opened / checkout_abandoned / purchase_failed, posted to the public diagnostics ingest endpoint). A build that does not emit them is invisible here, so every step is a FLOOR and a rising conversionPct can mean emitter adoption rather than a better paywall."*
> 2. *"Each step counts DISTINCT USERS inside its IST day, not event rows — a paywall that re-renders, or a checkout sheet reopened three times, is one user. Window `totals` sum those per-day distinct counts, so a user active on three days counts three times."*
> 3. *"auth_events rows with a NULL user_id (a pre-auth or logged-out emission) cannot be attributed to a person and are EXCLUDED from every step. They are not zero; they are unattributable."*
> 4. *"ordersPaid is server truth from `orders` (status PAID, is_test = false — iOS sandbox purchases hit production and are excluded from every money surface), bucketed by the order's own IST creation day. It is NOT joined to the events, so conversionPct can exceed 100% when someone buys without a recorded paywall view."*
> 5. *"conversionPct is NULL, never 0, on a day with no recorded paywall views — \"no denominator\" and \"converted nobody\" are different facts."*
> 6. *"auth_events rows are purged after 90 days by the nightly job, which is why the window caps there."*
> 7. *"auth_events rows carry NO exam of their own. With `exam` supplied, an event is attributed to the exam the user has picked AT QUERY TIME (user_profiles.active_exam_id, NULL ≡ upsc-cse) — not to the exam whose paywall they were actually looking at. …"* (continues with the `orders.exam_id` counterpoint quoted in the paragraph above)

**A constructed illustration of the two awkward shapes** — *not* a capture, because
staging has no paywall traffic to produce them, but both are reachable and both must
render as written:

```jsonc
{ "day": "2026-09-20", "paywallViewed": 2, "upgradeTapped": 1, "checkoutOpened": 1,
  "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 3, "conversionPct": 150 },
{ "day": "2026-09-19", "paywallViewed": 0, "upgradeTapped": 0, "checkoutOpened": 0,
  "checkoutAbandoned": 0, "purchaseFailed": 0, "ordersPaid": 1, "conversionPct": null }
```

**2026-09-20 converts at 150%** (three orders against two recorded paywall views —
emitter coverage, not a miracle), and **2026-09-19 converts at `null` with a paid order**
(nobody's build emitted a view that day). Caveats 4 and 5 above are the server saying so
in its own words.

⚠️ **`totals` are per-day distinct counts *summed*, not window-wide distinct users.** A user
who viewed the paywall on three days counts three times. That is the only total consistent with
the chart above it; a window-wide distinct count would not add up to the series.

---

## 5. How it's stored & bounded (the "no time-bomb" guarantee)

- **Capture** (`ApiUsageInterceptor`): one row per *authenticated* request — `userId, method, route TEMPLATE (no IDs/query/body/PII), status, durationMs, createdAt`. Batched + fire-and-forget, so it adds ~zero request latency. Public / `x-api-key` / webhook routes (no user) are skipped.
- **Rollup + purge** (BullMQ repeatable jobs, mirroring `payment-reconcile`): nightly `rollup` collapses raw rows into `api_usage_daily` (`day×user×route×count`); `purge` drops raw >30d and rollup >120d. So the tables are permanently bounded.
- **Plane A** reads the app's live tables directly — nothing to maintain or purge there.

## 6. Caveats
- `engagedDau` is a floor (browse-only sessions aren't counted); `activeDau` is the broader "any request" number once the capture has data.
- `paywall-hits` and `heatmap` are raw-sourced → ~30-day window; everything else honours the full `days`/date range (Plane A: months; Plane B rollup: 120d).
- IST always (`days` boundaries resolve in IST).
- **Exam (§0b):** without `?exam=` every number here is a **cross-exam total** — the app is multi-exam since 2026-08-25, so "users" means users of *all* exams. With `?exam=`, a PERSON-sourced number is retro-attributed to today's picked exam and a REQUEST-sourced number starts at `examCoverage.since`. Never put a per-exam number next to a pre-deploy all-exam number.
- **`?exam=` also changes SHAPE on four routes** (`dau`, `paywall-hits`, `heatmap`, `endpoints`: bare array → `{ data, examCoverage }`) and changes the **meaning of "premium"** on `premium-engagement`.

---

## 7. Auth diagnostics + api_usage client context (raw tables)

Two additive data sources landed for auth-failure triage and per-device attribution.
No SME endpoint reads them yet (that's a later module) — query the raw tables directly.

### `auth_events` — pre-auth + auth-lifecycle sink
Written by the **@Public** `POST /diagnostics/auth-events` ingest (mobile emitters).
The whole point is capturing failures *before* a user is authenticated (login / OTP /
refresh), so there is **no FK on `user_id`** (rows survive account deletion). The raw
login identifier is **never stored** — only `identifier_masked` (last-4 visible, e.g.
`********7935`) and `identifier_hash` (SHA-256 hex of the normalized value: email
lowercased, phone digits-only). To look up a phone, hash it the same way and match on
the hash.

`event_type` — auth lifecycle: `login_attempt | login_success | login_failed | otp_send_failed |
otp_verify_failed | refresh_failed | forced_logout`; general client events: `client_error |
paywall_viewed | upgrade_tapped | checkout_opened | checkout_abandoned | purchase_failed |
chat_conversation_started | app_opened | app_backgrounded`. The full list is
`src/modules/diagnostics/constants/auth-event-types.ts` — read it there, not here. The general
client events are emitted by app **2.0** (released 2026-09-17) and by no older build, so their
volume grows with adoption; `chat_conversation_started` is additionally written server-side by
`QuotaService`. Five of them — `paywall_viewed`, `upgrade_tapped`, `checkout_opened`,
`checkout_abandoned`, `purchase_failed` — now have a first-class endpoint: **§4c
`/sme/analytics/purchase-funnel`**. Rows with a **NULL `user_id`** (a pre-auth or logged-out
emission) cannot be attributed to a person and are **excluded** from every funnel step: they are
not zero, they are unattributable. (Validated at ingest, stored as text for forward-compat.) Other columns: `device_id`,
`platform`, `app_version`, `device_model`, `os_version`, `error_code`, `message` (≤500),
`ip`, `created_at`. Rate-limited 60/hour per device (fallback IP) — silent 429.

```sql
-- Failed logins in the last 7 days for a specific phone (hash it the same way the
-- server does: digits-only, then sha256 hex).
SELECT created_at, event_type, error_code, platform, app_version, ip
FROM auth_events
WHERE identifier_hash = encode(digest('919876547935', 'sha256'), 'hex')  -- pgcrypto
  AND event_type IN ('login_failed', 'otp_verify_failed', 'otp_send_failed')
  AND created_at > now() - interval '7 days'
ORDER BY created_at DESC;

-- Which device_ids has one user authed from? (correlate account sharing / churn)
SELECT DISTINCT device_id, max(created_at) AS last_seen
FROM auth_events
WHERE user_id = $1 AND device_id IS NOT NULL
GROUP BY device_id
ORDER BY last_seen DESC;
```

### `api_usage` context columns
`api_usage` gained nullable `app_version`, `platform`, `device_id` (from the
`x-app-version` / `x-platform` / `x-device-id` request headers). Row-level context
only — the `api_usage_daily` rollup carries none of those three (still `day × user × route`).
Populated deploy-day forward for header-sending clients; older rows stay null.

`api_usage` **and** `api_usage_daily` additionally gained nullable **`exam_id`** — the exam the
request was made in, as resolved by `ExamContextMiddleware` (`req.exam`; absence of the
`X-Exam` header ≡ `upsc-cse`). The rollup's unique key is now
`(day, userId, route, examId)`. This is the column behind every REQUEST-sourced `?exam=` filter
and behind `examCoverage` (§0b). It is stamped **forward only, from the 2026-09-21 deploy**: a
NULL means *"captured before the column existed"*, the row is excluded from every per-exam
number — `upsc-cse` included — and it is counted in `examCoverage.unknownRows`.

Going forward the interceptor never writes a NULL there (the middleware always resolves
something), so `unknownRows` is a purely **historical** quantity: it shrinks to zero on the raw
table once the 30-day retention window clears 2026-09-21, and on the rollup once 120 days do.
A large `unknownRows` today is expected and is not a capture failure.

```sql
-- Distinct devices per user from real API traffic (30d raw window).
SELECT user_id, count(DISTINCT device_id) AS devices
FROM api_usage
WHERE device_id IS NOT NULL AND created_at > now() - interval '30 days'
GROUP BY user_id
ORDER BY devices DESC;

-- App-version spread of active users (last 7 days).
SELECT app_version, platform, count(DISTINCT user_id) AS users
FROM api_usage
WHERE app_version IS NOT NULL AND created_at > now() - interval '7 days'
GROUP BY app_version, platform
ORDER BY users DESC;
```

---

Related: [[reference_usage_dau_sources]], [[sme-portal-integration]], [[feedback_ist_timezone]].
