# SME Portal — Metrics Decoder

> Changed 2026-09-21 — stale-text pass (telemetry live in 2.0, version ladder, links). Exam-dimension changes follow in [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md).
> Changed 2026-09-21 — ?exam= on every analytics route, examCoverage, purchase-funnel (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

**What this is:** one line per number on every SME screen — what it literally counts, and what it does *not*.
Written so you can answer "what is this number?" in a meeting without opening code.

**Audience:** us + the SME portal team.
**Source of truth:** `src/modules/sme/services/*` in this repo. Anything the portal computes itself is marked **[portal-side]**.
**Format:** the PDF export was removed on 2026-09-21; this markdown is the only maintained copy.

---

## 0. Read this first — why two screens can show different "premium" or "active" numbers

There are **three separate data planes**. Nothing reconciles them, and that is by design.

| Plane | Where it comes from | Covers | Retention |
|---|---|---|---|
| **A — Domain tables** ("engaged") | 6 real product tables: `simulation_attempts`, `user_content`, `user_question_attempts`, `custom_tasks` (only where `source_id IS NULL`), `psychometric_test_results`, `user_document_progress` | Only *genuine actions*. A user who opened the app and scrolled is invisible. | Full history |
| **B — Request capture** ("active") | `api_usage` (raw) + `api_usage_daily` (rollup) | Every **authenticated** API call. Public routes, `x-api-key` routes and webhooks are skipped. | **raw = 30 days**, rollup = long |
| **C — Billing/user tables** | `user_auth`, `orders`, `payments`, `payment_events` | Money + account state | Full history |

Rules of thumb:
- **engaged < active** almost always. Engaged is "did something", active is "made any request".
- Anything sourced from plane B **cannot** go back more than ~30 days. A low number in an old window is *retention*, not a quiet week.
- All day buckets are **IST (UTC+5:30)**. A "day" is midnight-to-midnight India time, not UTC.
- Chat, mains evaluation and PYQ-variation have **zero persistence** — they never appear in any usage number, anywhere.

### 0.1 The fourth axis of disagreement — **which exam** a number belongs to

The three planes above explain why two screens disagree about *when* and *how much*. Since
2026-09-21 every `/sme/analytics/*` route also takes `?exam=`, and that adds a fourth,
independent way for two correct numbers to disagree — because **"exam" is four different
columns** and no two of them answer the same question.

| # | "Exam" means | Column | True for | The disagreement it causes |
|---|---|---|---|---|
| 1 | **What the person is studying** | `user_profiles.active_exam_id` | **as of right now**, retroactively applied to their whole history | A June signup who switched to APPSC last week is a **June APPSC signup**. There is no switcher history and never was. |
| 2 | **What the person has bought** | a live row in `user_exam_entitlements` | per exam, exact | An APPSC subscriber who is *studying* UPSC is **premium in APPSC and free in UPSC**, both at once. |
| 2b | **The person-level premium mirror** | `user_auth.premium_expires_at` / `status` | the person, not the exam | Flips on **any** purchase. This is what "premium" means with **no** `?exam=`, so the un-filtered premium count is **≥** the sum of the per-exam ones. |
| 3 | **Which exam the request was made in** | `api_usage.exam_id` / `api_usage_daily.exam_id` | per request, **only from 2026-09-21** | Everything before that deploy is **NULL = unknown**, excluded from every exam. Per-exam usage history simply does not exist yet. |
| 4 | **Which exam the content is for** | `exam_ids` / `examIds` arrays (`*` = all) | per document | A `*`-tagged document counts for **every** exam, so per-exam content numbers sum **above** the total. |

Two NULL rules that **point in opposite directions**, which is the single most confusing thing
on this page:

* **NULL `active_exam_id` ≡ `upsc-cse`.** The person onboarded before app 2.0 (2026-09-17, which
  made the exam picker onboarding step 1) and has never used the home switcher — so the default
  is literally what they were served. This is still **most of the user base** and only shrinks
  as 2.0 is adopted. Matching the literal slug alone would report UPSC as having almost no users.
* **NULL `api_usage.exam_id` = "not recorded", and the row is EXCLUDED** — `upsc-cse` included.
  Here NULL means the request predates the column, not that it was UPSC.

**Practical consequence for a meeting:** with no `?exam=` every number in this document is a
**cross-exam total**. With `?exam=`, a person-sourced number is retro-attributed and a
request-sourced number starts on 2026-09-21. The two are not comparable, and the response tells
you which you are holding (`examCoverage` = request-sourced).

---

## 1. Analytics screen (`/analytics`)

Powered by `GET /sme/analytics` — but this endpoint is **built by the portal**, not by us. It pulls the full `/sme/users`, `/sme/orders` and `/sme/transactions` lists and adds them up in Python. **[portal-side]**

### KPI cards

| Card | What it is | Watch out |
|---|---|---|
| **Total Users** | Count of every row in `user_auth`. | ⚠️ Portal fetches at most **6,000 users** (100/page × 60 pages). Past 6,000 this card silently stops growing — and so does every other number on this screen. |
| **Premium Access** | Users with premium *access right now* = `SUBSCRIBED` and not expired, **OR** `ACTIVE` and still inside their trial window. | Includes trials. This is "who can use paid features", not "who paid". ⚠️ **The trial window is per exam, not 14 days** (APPSC Group 1 is currently 0) — corrected 2026-09-21; before that this number used a fixed 14 days for everyone and over-counted on any exam with a shorter trial. |
| **Paying Subscribers** | The `premiumState = 'Premium'` bucket = `SUBSCRIBED` and not expired. | The honest revenue-side number. Always ≤ Premium Access. |
| **Revenue (captured)** | Sum of `CAPTURED` payment amounts, in ₹, **excluding** `MANUAL` grants. | Manual comps are shown separately in "Revenue by source" so the headline isn't inflated by free grants. ⚠️ This sums the **payment** ledger, not paid orders — see the note below. |
| `x% onboarded` | Share of users with `user_profiles.onboarding_completed = true`. | Current state, not a cohort. |
| `x% of users` (on Premium Access) | Premium Access ÷ Total Users. | |
| `x% success` (on Revenue) | Captured payments ÷ all payment rows. | Denominator includes `CREATED`/`FAILED` attempts, so a user retrying three times drags it down. |

> **Paid orders vs payments — the check that caught a real bug (2026-07-29).**
> "Revenue (captured)" sums the **payments** table; "Order status → PAID" counts **orders**. They should agree. On 29 Jul they did not: 16 PAID orders against 14 payments. Two Razorpay purchases had been collected with no payment row, so ₹6,499 sat outside the revenue figure — and a refund on them could not have revoked premium, because `handleRefund` resolves the user through payment → order → userId. Root cause was ours, not the portal's: payment rows were only ever created by the client-called `/payments/verify`, both webhook handlers were update-only, and `verifyPayment` short-circuits on `premiumGrantedAt` — so a webhook-first purchase (routine on UPI intent, where the bank confirms before the user returns to the app) could never get a row. **Fixed** in `payments.service.ts` — both webhook handlers now call the idempotent `upsertCapturedPayment` — and **backfilled 2026-08-06** via `scripts/backfill-missing-razorpay-payments.sql`. The recurring-subscription path was never affected. Keep comparing these two numbers: it is the cheapest revenue-integrity check on the screen.

### Alert strips

| Alert | What it is | Watch out |
|---|---|---|
| **N paid order(s) need reconcile** | `PAID` orders where `premiumGrantedAt` is NULL. | ⚠️ **The column only shipped 2026-06-16.** Every order paid before that date is NULL and counts here forever. Only orders newer than that date are a real alert. |
| **N premium expiring within 7 days** | Currently-premium users whose `premiumExpiresAt` falls in the next 7 days. | Trial users have no `premiumExpiresAt`, so they never appear. |

### Bar lists

| Chart | Each bar is | Watch out |
|---|---|---|
| **Subscription funnel** | Count of users per `premiumState`: `Premium` (paid, live) · `Trial` (inside **that user's exam's** trial window — never hard-code 14) · `Trial Ended` (lapsed, never paid) · `Churned` (paid once, now expired) · `Downloaded` (signed up, trial still open but nothing else). | Derived from timestamps, not from the `status` column — so it's correct even for users who haven't opened the app since expiry. Exactly one bucket per user. |
| **User status** | The raw `user_auth.status` enum (ACTIVE / SUBSCRIBED / SUSPENDED / …). | This is the *stored* column and can be stale — `status` is only refreshed on an authenticated request. Prefer Subscription funnel. |
| **Sign-in provider** | Google / Apple / phone, from `user_auth.provider`. | |
| **Order status** | Orders by `CREATED` / `PAID` / `FAILED`. | An abandoned checkout leaves a `CREATED` row forever, so `CREATED` is usually the biggest bar and means nothing. |
| **Plan mix** | Orders by `planType` (MONTHLY / ANNUAL). | Counts orders, not people. An Apple annual renewal creates a new order each year. |
| **Payment method** | Captured payments by Razorpay method (`upi`, `card`, …) or `apple_iap`. | |
| **Revenue by source** | Captured ₹ split by the parent order's `paymentSource`: APPLE / RAZORPAY / MANUAL. | A payment whose order wasn't in the fetched page set lands in **"Unknown"** — a big Unknown bar means the 6,000-row cap bit, not a data problem. |

### Apple In-App Purchases block

| Number | What it is | Watch out |
|---|---|---|
| **Apple revenue** | ₹ on `PAID` orders with `paymentSource = APPLE`. | Includes renewals; each renewal is its own order. |
| **Subscribers** | Distinct `userId` across those paid Apple orders. | Lifetime distinct, not "currently subscribed". |
| **Paid orders (incl. renewals)** | Count of those orders. | |
| **Events recorded** | Rows in the `payment_events` ledger. | ⚠️ **Currently wrong — this counts Razorpay events too.** The portal asks for `?source=APPLE` but its HTTP client overwrites the query string with `page`/`limit`, so the filter is dropped. See §8. |
| **New / Renewals / Billing issues / Expired / Refunds** | Apple notification types: `SUBSCRIBED`+`OFFER_REDEEMED` / `DID_RENEW`+`RENEWAL_EXTENDED` / `DID_FAIL_TO_RENEW` / `EXPIRED`+`GRACE_PERIOD_EXPIRED` / `REFUND`+`REVOKE`. | These five *are* correct — they match on an `apple.` prefix, so the leaked Razorpay rows don't affect them. Only the "Events recorded" total is polluted. |

### Month bars

| Chart | Each bar is | Watch out |
|---|---|---|
| **New users by month** | Signups per calendar month, last 12 months. | Built from the fetched user rows — subject to the 6,000 cap. |
| **Revenue by month (paid orders)** | ₹ of `PAID` orders by the month the **order was created**. | Order-created, not payment-captured. A renewal lands in its own month. |

---

## 2. Engagement screen (`/engagement`)

Powered by `GET /sme/analytics/<metric>` — these **are** computed by our backend.

### Pulse

| Number | What it is | Watch out |
|---|---|---|
| **DAU (24h)** | Distinct users with ≥1 *genuine action* (plane A) in the last 24 hours. | Rolling 24h, not "today". |
| `active N` (under DAU) | Distinct users who made any authenticated request in 24h (plane B). | Always ≥ DAU. If it's null/0, request capture isn't reporting. |
| **WAU (7d)** / **MAU (30d)** | Same as DAU but 7 / 30 days. | |
| `active N` (under MAU) | Distinct users in `api_usage_daily` over 30 days. | |
| **Stickiness** | engaged DAU(24h) ÷ engaged MAU(30d). | The standard "how many monthlies show up daily" ratio. 10–20% is normal for a study app. |

### Daily active users chart
Two lines per IST day: **engaged** (plane A, full history) and **active** (plane B). The active line is **null before request capture shipped** — a gap on the left is missing capture, not zero traffic.

### Feature pull

| Number | What it is | Watch out |
|---|---|---|
| **actions** | Row count per feature in the window: `simulation`, `mcq`, `doc_reading`, `psychometric`, `ai_<type>`, `planner`. | `planner` counts only user-created tasks (`source_id IS NULL`), never auto-seeded ones. |
| **users** | Distinct users per feature. | Chat / mains-eval / PYQ-variation are **absent by design** — nothing is stored for them. |

### Monetization × engagement

| Number | What it is | Watch out |
|---|---|---|
| **premium / free rows** | Users split on `premium_expires_at > now`. | ⚠️ **A different definition of "premium" than the Analytics screen.** Trial users land in `free` here (no expiry date), and a churned-but-not-yet-expired user lands in `premium`. Expect this count to differ from both Analytics cards. |
| **activeUsers** | Of those, how many took ≥1 action in the window. | |
| **avg actions / active user** | totalActions ÷ activeUsers. | Denominator is *active*, not all — so it never gets diluted by dormant accounts. |
| **Paywall hits** (by route) | HTTP **402** responses per route, i.e. someone hit a paid wall. | Upgrade intent, not failures. From raw `api_usage` → **max ~30 days** regardless of the window you pick. |

### Retention & risk

| Number | What it is | Watch out |
|---|---|---|
| **Cohort size** | Users who signed up inside the window. | |
| **D1 / D7 / D30** | Of that cohort, how many were active **at any point ≥1 / ≥7 / ≥30 days after their own signup**. | ⚠️ **Cumulative, not "on day 7".** And someone who signed up 3 days ago *cannot* satisfy D7 — so a 30-day window always shows a depressed D7/D30. Compare cohorts of equal age, or widen the window. |
| **Churn risk** | Users whose last genuine action was more than `inactiveDays` ago **but within the last 90 days**. | ⚠️ Users who **never** did anything are excluded entirely, and anyone silent >90 days drops off the list. This is "going quiet", not "already gone". |

### Behaviour

| Number | What it is | Watch out |
|---|---|---|
| **Heatmap** | Count of **requests** (not users) per IST weekday × hour. | Plane B → last ~30 days only, whatever window you set. |
| **Top documents / readers** | Progress rows per document title in the window. | Grouped by title+subject, so two docs with the same title merge. |
| **Mock completion** | Simulation attempts submitted ÷ started in the window. | An attempt started on day 29 and submitted on day 31 counts as started-not-submitted. |
| **Notification read rate** | `notification_history` rows with `isRead = true` ÷ rows sent, in the window. | ⚠️ `isRead` means "this user opened the in-app feed at some point after", **not** "tapped this push". See §5. Topic broadcasts aren't in this table at all. |
| **Endpoints** (calls / users) | From the `api_usage_daily` rollup: total calls and distinct users per route, top 100. | |
| **Platforms** | Distinct users with an **active device token** per platform. | A user with both iOS and web is counted twice. This is device registrations, not installs or sessions. |

### Video · reels
Straight from the **Mux Data API**, not our database.

| Number | What it is | Watch out |
|---|---|---|
| **Total views / Unique viewers** | Mux's own counts over the timeframe. | |
| **Watch time** | Total watch time **including paused and stalled** time. | The "actually playing" sub-line is the honest one. |
| **Avg watch / view** | Watch time ÷ views. | |
| **Top videos** | Mux's `video_id` **is** our `playbackId`; titles are joined in from the reels list. | A reel with no title just shows its playback id. |

### User explorer

| Column | What it is | Watch out |
|---|---|---|
| **Actions 30d** | Rows in the 6 domain tables (plane A) for that user in the last 30 days. | ⚠️ The current UI tooltip says "screen opens, taps & other tracked events" — **that is wrong.** We do not track screen opens or taps at all. It is only the 6 tables. Worth fixing in the portal. |
| **Last seen** | `max(sessions.lastAccessedAt)` — last time a session token was used. | Closer to "last opened the app" than Actions 30d. |
| **Premium until** | `premium_expires_at`. | NULL for trial users and for never-paid users alike. |

### Exam-scoped keys (new 2026-09-21)

These appear **only** when a panel was fetched with `?exam=`. With no exam selected none of them
exist and every number on the screen is a cross-exam total (§0.1).

| Key | What it is | Watch out |
|---|---|---|
| **`examCoverage.since`** | The earliest request **carrying an exam** inside the window, on the routes sourced from `api_usage` (`summary`, `dau`, `paywall-hits`, `heatmap`, `endpoints`, `release-health`). | ⚠️ **This is the start date of the per-exam number, not of the product.** The column shipped on 2026-09-21; everything before it is excluded. An ISO instant on most routes, an **IST `YYYY-MM-DD`** on `dau` / `endpoints` (their source is the daily rollup). `null` = nothing in this window carries an exam at all → the panel is *"not recorded yet"*, not zero. |
| **`examCoverage.unknownRows`** | How many rows in **this exact window** were dropped for having no exam. | Not an error count and not a backlog. It is large today by construction and shrinks to zero as the pre-deploy rows age out (raw at 30 days, rollup at 120). |
| **`meta.clientErrorsExamScoped`** (release-health) | Always **`false`** when an exam filter is on. | ⚠️ `clientErrors` comes from `auth_events`, which has **no exam column and no relation to join one through**. So with `?exam=` that one column stays **app-wide** while `activeUsers` / `requests` beside it are narrowed — which makes **`clientErrorsPerActiveUser` an overstatement**, not a per-exam rate. Judge builds on `errorRateExcludingPaywallPct` when an exam is selected, and never compare crash rates across exams. |
| **`examSelected.cohortSignups` / `windowSignups`** (onboarding-funnel) | "N of M signups in this window are yours." `cohortSignups` is the filtered number the rest of the response describes; `windowSignups` is **every** signup in the window, all exams. | The two denominators are deliberately different. `byExam` is computed over the **whole** window, not the filtered cohort — a breakdown of a set already narrowed to one exam would be a single bar. |
| **`examSelected.explicitlySelected(Pct)`** | Window signups carrying a **non-NULL** `active_exam_id` — i.e. who came through app 2.0's picker or have used the switcher. | This is an **adoption** metric wearing a funnel's clothes. It rises with 2.0 uptake, not with product quality. |
| **`examSelected.nullCount` / `nullBucketedAs`** | Window signups who never picked (pre-2.0 onboarding, switcher unused). Folded into `upsc-cse` in `byExam` and in every other per-exam number in the API. | The reason `upsc-cse` is not empty. When this number is large, "UPSC" partly means "unclassified". |
| **`examSelected` is not a stage** | It is a **sibling key** of `stages[]`, never a sixth entry in it. | Deliberate: `stages` is indexed positionally by the portal, so inserting a stage would silently renumber every existing one — for a caller who never asked about exams. |
| **the `authEvents` block under `?exam=`** | Stays **app-wide**. | A pre-auth event has no user to join an exam through. So an exam-scoped funnel sits next to an un-scoped failure panel. Label it, or a reader will attribute every OTP failure in the country to one exam. |

### Purchase funnel (`/insights/purchase-funnel`) — new 2026-09-21

`GET /sme/analytics/purchase-funnel`. `days` 1–90 (default 30) — `auth_events` is purged at 90.

| Number | What it is | Watch out |
|---|---|---|
| **Paywall viewed / Upgrade tapped / Checkout opened / Checkout abandoned / Purchase failed** | **Distinct users** per IST day who emitted that `auth_events` type. Not event rows — a paywall re-renders. | ⚠️ **Client-emitted by app 2.0+ only**, through the public diagnostics ingest. A user on an older build is invisible, so **every step is a floor** and a rising funnel can be adoption rather than improvement. Rows with a NULL `user_id` are *unattributable* and excluded — not zero. |
| **Orders paid** | `orders` at `PAID`, `is_test = false`, by the order's own IST creation day. | The only **server-truth** number on the panel. It is not joined to the events. |
| **Conversion %** | `ordersPaid ÷ paywallViewed × 100`, one decimal. | ⚠️ **`null`, never `0`, on a day with no views** — "no denominator" and "converted nobody" are different facts, and the first usually means the day predates the emitting build. ⚠️ **It can exceed 100%, and that is information, not a bug**: someone bought without a recorded paywall view, so the number is measuring **emitter coverage**. Do not clamp it and do not "fix" it. |
| **Totals** | Per-day distinct counts **summed**. | A user who viewed the paywall on three days counts **three times**. It is the only total consistent with the chart above it; a window-wide distinct count would not add up. |
| **Per-exam conversion %** | Steps attributed by the user's **current** `active_exam_id`; `ordersPaid` by `orders.exam_id`. | ⚠️ A **retro-attributed denominator** with an **exactly-attributed numerator**. Read a per-exam conversion as directional only; the all-exams number (no `?exam=`) is the exact one. The service ships this sentence in `caveats` verbatim. |

---

## 3. Insights → Question quality (`/insights/question-quality`)

Finds **broken questions**, not hard ones. `GET /sme/content/question-quality`. Default window **90 days**, minimum **20 attempts**.

| Number | What it is | Watch out |
|---|---|---|
| **Attempts** | Content pool: **distinct users** who answered (one upserted row per user per question). Simulation pool: answered rows in submitted attempts. | Blank answers are counted as `skipped` and excluded from attempts. |
| **Wrong** | Content pool: the user's **latest** stored `isCorrect`. Simulation pool: `selectedOption` compared against the question's **current** key. | Someone who missed it, retried and got it right counts as **correct** — so wrong-rate is a *floor*. If a key was fixed later, old simulation answers are re-scored under the new key. |
| **Wrong rate %** | wrong ÷ attempts. | Raw, no confidence — do not rank on this. |
| **Guessing rate / chance %** | `1 − 1/optionCount` — 75% for a 4-option MCQ. | The bar a well-formed question can't fall below. |
| **Confident wrong-rate %** | Wilson 95% **lower** bound on the wrong rate. | Collapses toward 0 at low sample. "We are 95% sure it's at least this bad." |
| **Flag: `worse_than_chance`** | Confident wrong-rate is **above** the guessing rate. | A correct question cannot beat random guessing. This is a defect, near-certainly. |
| **Flag: `suspicious`** | Confident wrong-rate above 60% but still below the guessing rate. | Might just be hard. Second priority. |
| **Priority score** | `max(0, confidentWrongRate − chance) × attempts`. | ≈ "learners we're 95% sure this item misled *beyond* what guessing would have cost". A 90%-wrong item seen by 500 outranks a 100%-wrong item seen by 12. **0 on almost everything at low sample — that's correct, not a bug.** |
| **Contradicts key** | The most-picked option got more votes than the marked answer. | The single strongest "the answer key is wrong" signal. Check these first. |
| **Learners misled** (summary) | Sum of priority scores across the filtered set. | A blast-radius total, not a user count. |
| **Eligible** | Questions that cleared the minimum-attempts bar and were scored. | The denominator for Worse-than-chance / Suspicious. |

---

## 4. Insights → Release health (`/insights/release-health`)

Answers "should we force-update?". `GET /sme/analytics/release-health`. Default window **14 days, hard max 30** (that's the raw-data retention limit).

| Number | What it is | Watch out |
|---|---|---|
| **App version** | The `x-app-version` header on each request. | ⚠️ **`pre-1.7 (no x-app-version header)` is a real cohort, not missing data** — the header shipped with 1.7, so every older build lands there. Never delete or merge that row. Current store build is **2.0** (Android 27, iOS 2.0 build 3, released 2026-09-17), so expect rows up to 2.0; `AppConfig.minAppVersion` on production is **1.0.0**, i.e. no forced update is in effect. |
| **Active users** | Distinct users who made a request on that build × platform. | A user on two builds in the window counts in **both** rows. |
| **Requests** | Captured authenticated requests for that build. | Public/login routes are not captured — **a build that can't get past login is invisible here.** Its signal shows up in Client errors. |
| **Error rate (excl. paywall)** | (4xx + 5xx − 402) ÷ (requests − 402). | **This is the number to judge a build on.** |
| **Error rate (incl. paywall)** | 4xx + 5xx ÷ requests, 402 included. | 402 is the paywall doing its job — a monetisation signal, not a fault. Ignore this column when comparing builds. |
| **Client errors** | `auth_events` rows of type `client_error` reported by the app itself. | Can create a row for a build that made zero successful requests. That's deliberate — a crash-only build is exactly what we're hunting. |
| **Client errors / active user** | clientErrors ÷ activeUsers. | The crash-proxy. |
| **Adoption share %** | This build's active users ÷ all active users on that platform. | ⚠️ **Can sum past 100%** across builds, because a user upgrading mid-window counts on both. |
| **vs. previous build** | `better` / `worse` / `similar` / `insufficient_data` against the next-older build on the same platform. | Thresholds: **±1.0 error-rate points** or **±0.5 client errors per user**. A verdict needs **≥200 requests and ≥5 active users on both builds** — otherwise `insufficient_data`, which means "too quiet to judge", not "fine". |
| **Not derivable** (headline) | Shown when there isn't a comparable pair. | |

---

## 5. Insights → Notification effectiveness (`/insights/notification-fx`)

`GET /sme/analytics/notification-effectiveness`. Default window **30 days**, maturation **3 days**, kill threshold **≤25%**, min matured sent **50**.

> **The one caveat that must travel with every number here:**
> `isRead` = *"this recipient opened the in-app notification feed at some point after this row was created."* It is set **in bulk** for all of a user's unread rows when they open the feed. It does **not** mean they tapped this push, saw this push, or that the push was even delivered — **push delivery is not tracked at all.** Read it as "did this person come back to the app", not "was this notification compelling".

| Number | What it is | Watch out |
|---|---|---|
| **Sent** | Personal notification rows created in the window. | ⚠️ **Broadcasts are not here.** Topic sends live in `topic_notifications`, which has no read flag — they cannot be measured. |
| **Read** | Of those, how many have `isRead = true`. | See the caveat above. |
| **Matured sent / Matured read rate** | Only rows **older than the maturation window (default 3 days)**. | A push sent 20 minutes ago hasn't had a fair chance. **All rates are quoted on the matured sample only** — this is why recent days look empty. |
| **Pending** | Rows too young to judge. Excluded from every rate. | If Matured read rate says "maturing", everything in the window is still pending. |
| **Wasted sends** | maturedSent − maturedRead. | Blast radius. The kill-list is ranked by this, not by rate — killing a bad type that goes to 5 people saves nothing. |
| **Recipients** | Distinct users who got that type. | Built from the newest 20,000 rows only; the banner says so when the cap is hit. |
| **Types** | Distinct `type` values in the window. | |
| **Worst performers** | Types with ≥50 matured sends **and** ≤25% read rate. | Tune both thresholds in the toolbar. Empty ≠ healthy — check "Matured sent" first. |
| **Time to read** | Always **"not derivable"**. | `read_at` exists but records the *bulk feed open*, not this notification's open. Any number here would be fabricated, so we return none. |

### 5.1 There is now an honest alternative — but only for engine-sent notifications

`GET /sme/notification-rules/:id/analytics` returns `sent` / `tapped` / `tapRate` from
`notification_dispatch`. **`tapped` is set only when the user actually opened that
notification** — the thing `isRead` was never able to tell you.

Three caveats, all load-bearing:

- It covers **only notifications sent by a rule**. Ad-hoc sends (`/sme/notifications/broadcast`,
  `/segment`, `/sme/users/:id/notify`) write no dispatch row and are still `isRead`-only.
- **`tapped` is client-reported, and only app 2.0 and later report it.** The server side is
  live (`POST /notification-preferences/tapped` stamps `notification_dispatch.tapped_at`);
  app **2.0** shipped to both stores on **2026-09-17** and is the first build expected to call
  it, which the backend cannot itself prove — confirm in the data. So `tapRate` describes the
  2.0-adopted share of a rule's recipients and climbs with adoption. A low rate means "partly
  measured", not "nobody tapped". Do not kill a rule on it.
- **Delivery is still not tracked.** A tap proves delivery; the absence of one does not prove
  non-delivery. `tapRate` remains a floor, not a rate.

Never place `tapped` and `isRead` side by side without labels — `isRead` is much larger by
construction, and the comparison invites exactly the wrong conclusion.

**How to read a low rate honestly:** a type sent mostly to already-engaged users scores well regardless of its content. A **very low** rate is the trustworthy direction of this signal; a high rate is weak evidence.

---

## 6. Insights → Onboarding funnel (`/insights/onboarding-funnel`)

`GET /sme/analytics/onboarding-funnel`. A **signup cohort**: everyone who signed up in the window, followed to their current state.

| Stage | What "reached" means |
|---|---|
| **1. Signed up** | `user_auth.createdAt` in the window. **This is the denominator.** |
| **2. Phone verified** | `phone_verified = true` right now. |
| **3. Onboarding completed** | `user_profiles.onboarding_completed = true` right now. |
| **4. Psychometric resolved** | Completed **OR** explicitly skipped **OR** a result row exists. It's "past the gate", not "took the test" — the `completed / skipped only / with result row` mini-tags break it down. |
| **5. First genuine action** | First row in any of the 6 domain tables at or after signup. |

| Number | What it is | Watch out |
|---|---|---|
| **% of signups** | reached ÷ cohort signups. | |
| **Conversion from previous** | reached ÷ previous stage's reached. | |
| **Dropped / drop-off %** | previous.reached − reached. | ⚠️ **Can be negative.** Some users satisfy a stage without the one before it (e.g. finished onboarding without a verified phone). We report it raw rather than clamping, and `reachedWithoutPrevious` tells you how many. |
| **Median / p90 time to reach** | Nearest-rank percentile, so it's always a duration someone really experienced. | **Stage 3 has no timing at all** — there is no `onboarding_completed_at` column, and inventing one from `updated_at` would be a lie. Stage 4 times only the *completed* path; a skip has no timestamp. |
| **Sample coverage %** | Share of people at that stage who carry a usable timestamp. | Low coverage = the median describes a minority. |
| **First action `bySource`** | Which of the 6 tables produced the first action. | ⚠️ `psychometric_test_results` is in that union — so a user whose only action was the psychometric test satisfies **stage 5 with the same row that satisfied stage 4**. This breakdown is how you see how many. |
| **Auth events** (login failed / otp send failed / otp verify failed) | App-wide pre-auth failures for that IST day, from `auth_events`. | ⚠️ **Capture began 2026-07-23** — days before that show `not rec.`, never 0. And coverage is **split**: `otp_send_failed` is emitted server-side and is **real**; `login_attempt / login_success / login_failed / otp_verify_failed` are **client-emitted by app 2.0 (released 2026-09-17) and by no older build**, so they are a **floor** that grows with 2.0 adoption. A low number there means "partly recorded", **never** "no failures". |
| **Login success rate** | login_success ÷ login_attempt for the covered days. | Both terms come only from 2.0 devices, so the ratio describes the 2.0 cohort, not the install base (see above). |
| **Worst-day correlation** | The covered day with the worst phone-verification rate, with that day's OTP failures beside it, plus the median OTP-verify failures for comparison. | Needs a day with **≥5 signups**. Its whole purpose: put an OTP spike next to the verification drop it caused. |

---

## 7. Other screens (quick reference)

| Screen | Number | What it is |
|---|---|---|
| **App Users** | `premiumState` badge | Same 5-bucket funnel label as the Analytics chart. |
| | `trialDaysLeft` | Days to the **effective** trial end (`trialEndsAt` if an SME extended it, otherwise signup + 14 days). |
| | Platform filter | Filters on **active device tokens**, not on sign-in provider. **[portal-side]** The portal's local filter compares against `provider`/`subscriptionSource` instead — different meaning, different results. |
| **Transactions** | Amounts | Always **₹**. We divide the stored paise by 100 before sending. |
| | `premiumGrantedAt` on an order | When premium was actually granted. NULL on every order older than 2026-06-16. |
| **Feedback** | Chat thumbs summary | Only rows the app explicitly submitted; the AI conversation itself is not stored. |
| **User detail → Timeline** | Merged record across 17 sources | Reverse-chronological, cursor-paginated. Absence of an event means "not captured", not "didn't happen". |

---

## 8. Known-wrong / misleading numbers (the short list)

**Status re-verified live against portal build v2.32.0 on 2026-07-29.** That release closed most of the labelling items the same morning. Verify against the live portal, never against a downloaded copy of the portal repo — the snapshot in `~/Downloads` carries no git history and goes stale within days.

### Still open

1. **Analytics → Apple "Events recorded" is inflated with Razorpay events.** The portal requests `/sme/webhooks?source=APPLE`, but its HTTP client replaces the whole query string with `page`/`limit`, dropping the filter. **Proven:** the webhook ledger holds 30 events in total, its first row is `eventType: order.paid, source: razorpay`, and the Apple panel reports exactly 30. **[portal-side]** — pass `source` as a param rather than baking it into the path. The five lifecycle counters below it match on an `apple.` prefix and are unaffected.
   ⚠️ Do not "prove" this by arithmetic: 3 + 11 + 3 = 17 against a total of 30 does **not** demonstrate a leak, because other legitimate `apple.*` types would explain the same gap. The proof is a razorpay row inside the ledger being counted.
2. **Analytics is capped at 6,000 rows per list.** Total Users, revenue totals, signups-by-month and every bar on that screen quietly stop growing past that; failed pages are silently dropped too. Not biting yet — 3,989 users at ~68/day — but it starts truncating around **late August 2026**. **[portal-side]** — raise the page cap or move the aggregation to our backend.
3. **Plane-B cards still over-promise their window.** Release Health is now capped to 7/14/30 and the heatmap carries a note, but **paywall hits still offers a 7/30/60/90 selector over ~30 days of data**.
4. **The Onboarding Funnel's stage-5 breakdown renders `[object Object]` and `NaN%`** — the portal is stringifying an array of objects. Still present on v2.32.0. The 1,079 headline above it is sound.

### Misleading by construction (not defects — read them before quoting)

These four are **working as designed**. They are listed here because each one will be reported
as a bug, and each one has a correct sentence that must travel with it.

| # | The number | Why it misleads | What to say instead |
|---|---|---|---|
| A | **Every analytics number with no `?exam=`** | It is a **cross-exam total**. The app has sold APPSC since 2026-09-08 and shipped multi-exam on 2026-08-25, so an MAU figure quoted off this screen is MAU across all exams — and the un-filtered "premium" count uses the **person-level mirror**, which flips on any purchase. | Say "…across all exams" out loud. If someone wants UPSC specifically, pass `exam=upsc-cse` — and then read row B. |
| B | **Per-exam usage history before 2026-09-21** | It does not exist. `api_usage.exam_id` was added that day; earlier rows are NULL and are **excluded**, UPSC included. A per-exam DAU chart therefore *starts* on deploy day. | "Per-exam request data starts 21 Sep 2026 — `examCoverage.since` says so on the response." **Never** compare a per-exam number with a pre-deploy all-exam number; the older one contains every exam. |
| C | **Any per-exam number sourced from `active_exam_id`** (signups, retention, churn, features, platforms, notifications, the funnel cohort) | **Retro-attribution.** The user is attributed to the exam they have picked *today*, applied backwards over their whole history. There is no switcher history and the schema has nowhere to keep one. | "Attributed to each user's current exam." Directionally useful, exactly wrong for anyone who switched. Worst on a **new** exam, where switchers are a large share. |
| D | **`clientErrors` / `clientErrorsPerActiveUser` on an exam-filtered release-health** | `auth_events` has no exam column, so that column stays **app-wide** while `activeUsers` beside it is narrowed. The ratio is an **overstatement** by roughly the inverse of the exam's share of traffic. | `meta.clientErrorsExamScoped: false` is on the response; surface it. Judge exam-filtered builds on `errorRateExcludingPaywallPct`, and compare crash rates only with the exam filter **off**. |

### Closed

5. ~~**"Paid orders need reconcile" is historically inflated**~~ — `premiumGrantedAt` only exists from 2026-06-16. Addressed by labelling: the Orders tab now explains that an order predating the field can look unreconciled even though premium was granted.
6. ~~**Three different "premium" numbers across two screens**~~ — the Engagement screen now states outright that its tier means "premium expiry > now, so active trials count as free — a different rule from Analytics' two premium figures".
7. ~~**"Actions 30d" claims screen opens and taps**~~ — replaced with the actual list of counted features, plus an explicit "screen opens and taps are not tracked".
8. ~~**D7/D30 retention looks broken on a short window**~~ — the screen now says so itself: "Anyone who signed up fewer than N days ago cannot qualify yet but still counts in the cohort … Not a broken metric."
9. ~~**Auth-event zeros in the funnel**~~ — capture is live and reporting (140 attempts / 117 successes on 29 Jul); pre-capture days render as "not rec." rather than 0.
10. ~~**Two Razorpay payments missing from the ledger**~~ — ours, not the portal's. Fixed in `payments.service.ts` and backfilled 2026-08-06. See the note under §1.
