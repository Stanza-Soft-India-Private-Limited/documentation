# SME Portal — what changed, 2026-09-21

This release puts an **exam dimension** on every module of the `/sme/*` surface. Almost all
of it is additive — an optional `?exam=` that, when omitted, returns exactly what the route
has always returned. **Two calls are not additive and will start returning `400` on the
production deploy: see §1.**

Backend commits in this release: `6baea4a` (schema) · `4f6622e` (users/orders/offers/banners)
· `bfae389` (content ingest) · `4cfe6c0` (analytics) · `c0a8fe5` (push/surveys/feedback) ·
`f6329e6` (per-exam `isPremium` + back-compat script).

**Production deploy date: `2026-09-21`.** Nothing below is live on
production until then. Confirm for yourself with the keyless probes in §9.

Every contract stated here was read off the controllers, DTOs and services in the release
tree, and **on 2026-09-21 every example in every module doc was re-run against staging**,
which is now running this exact build (`f6329e6`). A JSON block carrying
`<!-- captured from staging 2026-09-21, backend f6329e6 -->` is a **verbatim response**
(long arrays trimmed, always with a note saying so).

**Identifiers in examples are redacted; shapes and values are real staging captures from
2026-09-21.** Emails, names, phone numbers, support codes, user ids and provider
(order/payment/settlement) ids have been replaced with stable placeholders — consistently,
so cross-references between docs still line up. Nothing else was touched: every key name,
type, null, count, timestamp, exam slug and ordering is exactly as the server returned it.

A block still labelled *"shape verified against code"* is one staging **cannot** produce,
and each now says which of these it is:

| Why it could not be captured | Where |
|---|---|
| Staging's Firebase credential cannot mint an access token (500, `iam.serviceAccounts.getAccessToken`), so no push can be sent | `broadcast` / `segment` success bodies — `SME_NOTIFICATIONS_API.md` §2, §2.1, §3, §3.1; `SME_PORTAL_API.md` §4.1, §4.2. **Verified on the production credential path 2026-09-17 for the all-users case.** Every validation path on those routes *was* captured |
| Staging's Neo4j is unreachable (500, `Neo4j is unavailable (driver not initialised)`) | `GET /pyq/:id/details`, `GET /pyq/:id/reveal` — `SME_EXAMS_API.md`. Visible in an SME response only as `snapshot.streak.available: false`, which **was** captured |
| The data does not exist on staging | Apple rows (`SME_APPLE_IAP_API.md`), offer campaigns (`SME_OFFERS_API.md`, `SME_CAMPAIGN_ABANDONED_API.md`), feedback reports and survey responses (`SME_FEEDBACK_API.md`). The **envelopes** were captured empty; only the row shapes are from code |
| The write was destructive or out of scope | premium revoke, trial-extension success, campaign create, bulk content writes, `assign-questions` (it replaces a live mock test). Every one of these has its **refusal / validation response captured** instead |

🔴 **Three examples turned out to be wrong when run, and are corrected in place:** the
`POST /sme/content/pyq` body was missing three required fields; `PATCH
/sme/content/pyq/:id {"isVerified": true}` is a 400 (the field is not updatable at all);
and `POST /sme/content/mains` rejects `marks`. See `SME_CONTENT_INGEST_API.md` §3.1, §3.3
and §3.5. A fourth divergence — `isPremium` disagreeing with `premiumState` — was a
**backend defect, not a doc error**; it is fixed in this release (§2.6), and the capture
that exposed it is kept, annotated, at `SME_PORTAL_API.md` §1.1.

---

## 1. 🔴 BREAKING — read first

Two calls the portal makes today will start failing. Both are in the money path, and both
fail **loudly** (a `400` with a single-string `message`, not the array a validation-pipe
failure produces), so a handler that matches on one sentence will work.

### 1.1 `DELETE /sme/users/:id/premium` now requires `?exam=`

```
exam is required: pass ?exam=<examId> to revoke one exam, or ?exam=all to revoke every exam.
```

**What to change on the portal.** The revoke button must decide, and say, *which* exam it is
revoking. Pass `?exam=<slug>` for one exam, or the literal `?exam=all` to reproduce the old
behaviour. An unknown slug (anything other than `all`) is also a `400`.

**Why it is not defaulted.** Exams are sold independently. The bare call used to revoke
everything, which was the honest reading while there was one thing to own; today the same
call, made to refund one ₹499 APPSC purchase, silently destroys the same person's ₹4,999
UPSC subscription. There is no safe default, so there is no default.

The response now echoes what it did: `{ success: true, examId: "appsc-group-1" | null,
expiredAt }` — `null` only on the `all` path. Revoking one exam from someone who still holds
another correctly leaves them `SUBSCRIBED`; the audit row records the exam rather than
asserting `UNSUBSCRIBED`.

### 1.2 `POST /sme/offers` now requires `examId`

```
examId is required (e.g. upsc-cse). Campaigns are per exam.
```

**What to change on the portal.** The campaign create form needs an exam selector, populated
from `GET /sme/exams`, and it must send `examId`. An unknown slug is a `400` as well —
`offer_campaign.exam_id` has no foreign key, so a typo used to be accepted, reported as
success, and then never resolve for any request: an invisible campaign that looks live.

**Why.** A campaign is a per-exam object. Defaulting to `upsc-cse` files an APPSC campaign
under UPSC — priced against UPSC's list, locked against UPSC's one-live-campaign window, and
invisible to every APPSC paywall.

⚠️ **`PATCH /sme/offers/:id` is deliberately unchanged.** There `examId` is tri-state and an
omission means "leave it alone", which is unambiguous. Only create is strict.

### 1.3 Also strict, but not breaking: unknown slugs

Every new `?exam=` parameter validates its slug and returns `400` rather than an empty page —
on a management screen "0 rows" and "you typo'd the slug" look identical and only one is
worth acting on. Routes affected: `/sme/users`, `/sme/orders`, `/sme/offers`, `/sme/banners`,
`/sme/users/:id/incidents`, `/sme/feedback/reports`, every `/sme/analytics/*`, and both
notification sends. `/sme/orders` additionally accepts the literal `?exam=none` (see §4).

---

## 2. Fixed defects you should know about

These are behaviour changes that are **corrections**. If the portal has worked around any of
them, remove the workaround.

### 2.1 Segment push with `exam` dropped the tier predicate

`POST /sme/notifications/segment` with `segment=premium|trial` **and** `exam=upsc-cse` built
its where-clause by spreading the exam filter into the query. For the default exam that filter
is itself an `OR` (the NULL rule in §3), so it **replaced** the sibling `OR` key the tier test
lived in: `premium` lost the "not expired" half and pushed to lapsed subscribers; `trial` lost
the in-window test entirely, leaving `status: ACTIVE`, which is **every free account** — a
UPSC-scoped "trial" push reached the whole free base. The exam clause is now an `AND` arm,
which cannot collide with a sibling key. `free` was always correct and goes through the same
helper now. The smoke test that asserts this numerically is
[rollout-and-verify.md](./rollout-and-verify.md) §8.2.

### 2.2 `GET /sme/offers?exam=` used to `400`

It was documented as a known gap in [SME_OFFERS_API.md](./SME_OFFERS_API.md). It is wired now,
validated the same way as the write paths, and every row carries its own `examId`. Listing is
cross-exam by default.

### 2.3 `GET /sme/banners?exam=<typo>` used to return an empty list

It now `400`s. Same reasoning as §1.3 — an empty carousel and a misspelled slug must not look
the same.

### 2.4 `/cms/*` stored `examIds` un-normalised

`[" APPSC-Group-1 "]` was stored as that literal padded string, which matched no filter: the
content was invisible in its own catalogue with no error anywhere. `/cms/*` and
`/content-doc-admin` now trim, lower-case and de-duplicate, and additionally **consult the
exam catalogue and log a WARN** for a well-formed slug that names no exam (`appsc-grp-1`).

⚠️ It WARNs, it never `400`s, and the row is still stored with the bad tag — every import
script in existence calls `/cms/*`, and rejecting would break all of them at once for a class
of error they have been getting away with. Only `/sme/content/*` refuses (§10).

### 2.5 `bonusDays` was documented as honoured. It is not

Any non-zero `plans[].bonusDays` is rejected with

```
plans[i].bonusDays must be 0 — bonus days are not honoured by any checkout path yet, so a non-zero value would advertise free time the customer never receives.
```

Campaign purchases are one-time Razorpay orders and Apple offer codes; neither adds
entitlement days, and Apple cannot combine a discount with free months at all. **Delete the
`bonusDays` control from the campaign form** —
[SME_CAMPAIGN_FORM_GUIDE.md](./SME_CAMPAIGN_FORM_GUIDE.md) says where.

### 2.6 Person-level `isPremium` ignored the exam's trial length

`isPremiumUser()` took no `trialDays` argument. For a `status = ACTIVE` user with
`trial_ends_at` NULL it fell back to `created_at + 14 days` — the historic UPSC default —
**whatever exam the person was on**. `derivePremiumState()` had already been given the
per-exam length, so the two disagreed on exactly the users in the gap.

Where it showed up:

| Surface | Wrong answer before the fix |
|---|---|
| `GET /sme/users` / `GET /sme/users/:id` | `isPremium: true` beside `premiumState: "Trial Ended"` and `entitlements: []` |
| `GET /sme/users/:id/snapshot` | same, in `entitlement.isPremium` |
| `tier_premium` FCM topic | a lapsed trialist subscribed to the premium audience and received its pushes |
| payments / notification audience | same helper, same mis-answer |

**On APPSC Group 1 — `trialDays: 0` — that meant every new free signup read as premium for
their first 14 days.** UPSC was never affected: its trial *is* 14 days, so the fallback was
accidentally right, which is why this survived unnoticed.

Fixed by threading the exam's `trialDays` into `isPremiumUser(user, trialDays)` at every
call site. `isPremium` now agrees with `premiumState` and `entitlements[]`; devices pick up
the corrected topic on their next `GET /notifications/topics` sync, with no app release.

⚠️ **The captured example in [SME_PORTAL_API.md](./SME_PORTAL_API.md) §1.1 is annotated as
a pre-fix capture** and deliberately still shows `isPremium: true` — it was taken minutes
before the fix landed and is the clearest picture of the defect. On the released build that
row returns `isPremium: false`.

**Portal impact: none required.** `isPremium` is person-level by design (true for *any*
exam), so a per-exam badge should read `entitlements[]` either way.

---

## 3. The exam model in five paragraphs

**There are two axes and they are not the same question.** *Which exam is this person in?* is
`user_profiles.active_exam_id` — what they picked, one value at a time, changeable from the
home switcher. *What does this person own?* is `user_exam_entitlements` — one row per exam
they hold, each with its own `accessTier`, `status` (`LIVE` / `EXPIRED` / `REVOKED`), source
and expiry. A user can be *in* UPSC and *own* APPSC. Picking the wrong axis silently answers
a different question, so every route in this release names the one it uses.

**`user_auth.isPremium` / `user_auth.status` is a person-level mirror, not an entitlement.**
It flips to `SUBSCRIBED` on any purchase in any exam. That is correct for a person-level
question ("has this person ever paid?") and wrong for a per-exam one, so the read surfaces
switch behaviour on `?exam=`: `GET /sme/users?premium=true`, `GET /sme/analytics/users` and
`GET /sme/analytics/premium-engagement` all use the **mirror** when `exam` is absent and a
**live un-revoked entitlement row in that exam** when it is present. Without that switch,
every APPSC buyer would be listed as a premium UPSC user. `premiumExpiresAt` stays the mirror
in both modes, and no key is added or removed — only the meaning of `isPremium` changes.
Grant, revoke and trial extension are also split by intent: a **grant** is per exam, a
**trial extension** is person-level and spans every exam (the response names the exam whose
trial policy produced the window), and a **revoke** now demands an exam (§1.1).

**NULL means two opposite things and you must not harmonise them.** For the *picked* exam,
**NULL ≡ `upsc-cse`**: a NULL `active_exam_id` means the person onboarded on a build older
than app 2.0 and has never touched the switcher, so UPSC is what they were actually served —
and that is still the majority of the base. For the columns this release stamps **going
forward**, NULL means **unrecorded** and the row is **excluded**, `upsc-cse` included. That
applies to `feedback_reports.exam_id`, `api_usage.exam_id` / `api_usage_daily.exam_id`, and
`orders.exam_id` — where a NULL on a PAID order is surfaced as the `UNRESOLVED_EXAM`
reconcile bucket (§4). Backfilling any of these to `upsc-cse` would be a guess presented as a
fact; the fix is to say when the history starts, not to invent it.

**Content is a third axis: an `examIds` array, with `*` meaning every exam.** A content row
is visible to an exam when `examIds` overlaps `[<exam>, "*"]`. The `*` sentinel exists so that
material genuinely shared across exams does not have to be re-tagged every time an exam
launches — a data migration per launch, forever. One consequence to design for: per-exam
content numbers can sum **above** the all-exams number, because a `*`-tagged row counts for
every exam. (Request-sourced numbers sum **below** it, because unknown-exam rows are
excluded.) Simulation **questions** have no `examIds` of their own — the container carries the
exam and questions inherit it from the test they are assigned into.

**App 2.0 (both stores, 2026-09-17) is what makes the picked exam real, and it also caps
several numbers.** It added the exam picker as onboarding step 1, the home switcher, and
`X-Exam` on requests; `GET /config` now returns per-exam `features`. Before 2.0 nobody could
pick anything, which is why NULL is the common case and why the NULL-≡-UPSC rule exists.
Because there is **no history of `active_exam_id`** — no switch log — every person-keyed
per-exam number is **retro-attributed**: a user, and everything they ever did, is counted
under the exam they have picked *today*. A June signup who switched to APPSC last week is an
APPSC signup in June. For `upsc-cse` this is nearly a no-op (it absorbs every NULL); for a
newly-launched exam it is a one-directional overstatement that grows as people switch. Say so
on the panel; there is nothing to fix client-side.

---

## 4. Per-module table

| Module | What changed in the API | What you must change on the portal | Doc |
|---|---|---|---|
| **Users** | `?exam=` on `GET /sme/users` (narrows page **and** `total`, and changes what `premium=true` means — mirror vs live entitlement in that exam). List and detail rows gain `activeExamId`, `trialExamId` and `entitlements[]` (`{ examId, accessTier, status, source, expiresAt, isTrial, trialEndsAt }`). `POST …/premium` takes `examId`. 🔴 `DELETE …/premium` **requires** `?exam=` | Add the exam filter; render `entitlements[]` instead of a single premium boolean; **fix the revoke call (§1.1)** and make the operator choose an exam | [SME_PORTAL_API.md](./SME_PORTAL_API.md) · [SME_TRIAL_EXTENSION_API.md](./SME_TRIAL_EXTENSION_API.md) |
| **Support / activity trail** | `snapshot` gains `quota.examId`, `entitlement.activeExamId` and `entitlement.entitlements[]`; `timeline` items gain `exam` (set for the three sources that record one — `orders`, `api_usage`, `feedback_reports` — `null` everywhere else); `incidents` takes `?exam=`, which narrows the `api_usage` half only | Show which exam a timeline row belongs to; when `?exam=` is set on incidents, label that pre-deploy `api_usage` rows are excluded, not absent | [SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) · [SME_USER_DETAIL_PAGE_GUIDE.md](./SME_USER_DETAIL_PAGE_GUIDE.md) |
| **Transactions & orders** | Order rows carry `examId` (nullable) and a computed **`reconcileHint`**: `OK` (granted, or not PAID) · `GRANT_PENDING` (PAID, exam known, no `premiumGrantedAt` — re-run the grant) · `UNRESOLVED_EXAM` (PAID, no grant, exam NULL). `?exam=` filters orders; the literal `?exam=none` selects exactly the unresolved bucket. `/sme/webhooks` rows carry `examId` but are **not** filterable | Replace any client-side "PAID but not granted" derivation with `reconcileHint`. **New rule: never grant an `UNRESOLVED_EXAM` order** — Apple's payloads carry no exam, so this is a real charge for a product with no `exam_plans` row. Seed the plan first; granting without deciding the exam grants the wrong exam for real money | [SME_PORTAL_API.md](./SME_PORTAL_API.md) · [SME_APPLE_IAP_API.md](./SME_APPLE_IAP_API.md) |
| **Offers / campaigns** | 🔴 `examId` **required** on create (§1.2); `?exam=` on the list now works (§2.2); every row carries `examId`; the one-live-campaign rule is applied **per exam**; `bonusDays` must be 0 (§2.5) | Add the exam selector to the create form and to the list filter; show which exam a campaign belongs to; delete the `bonusDays` control | [SME_OFFERS_API.md](./SME_OFFERS_API.md) · [SME_CAMPAIGN_FORM_GUIDE.md](./SME_CAMPAIGN_FORM_GUIDE.md) · [SME_CAMPAIGN_ABANDONED_API.md](./SME_CAMPAIGN_ABANDONED_API.md) · [SME_CAMPAIGN_ANALYTICS_PAGE_GUIDE.md](./SME_CAMPAIGN_ANALYTICS_PAGE_GUIDE.md) |
| **Banners** | `?exam=` on the list is validated — unknown slug is `400`, not an empty carousel (§2.3); `examId` on create must be an existing exam | Handle the `400`; a banner belongs to exactly one exam and the form should say so | [SME_BANNERS_API.md](./SME_BANNERS_API.md) |
| **Analytics** | `?exam=` on **all 17** pre-existing `/sme/analytics/*` routes. Four of them change shape under it — `dau`, `paywall-hits`, `heatmap`, `endpoints` go from a bare array to `{ data, examCoverage }`. `examCoverage` = `{ since, unknownRows }` and appears wherever `api_usage` / `api_usage_daily` is the source | Add one global exam selector (All / per exam from `GET /sme/exams`); normalise with `Array.isArray(res) ? res : res.data`; and **render a "since &lt;date&gt;" label** on every panel that returns `examCoverage`. **Never compare a per-exam number against a pre-deploy all-exams number** — the history is unknown, not UPSC, so draw a start date rather than a cliff | [SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) §0b · [SME_ANALYTICS_FRONTEND_GUIDE.md](./SME_ANALYTICS_FRONTEND_GUIDE.md) · [SME_METRICS_DECODER.md](./SME_METRICS_DECODER.md) |
| **Purchase funnel** (new) | `GET /sme/analytics/purchase-funnel` — paywallViewed → upgradeTapped → checkoutOpened → checkoutAbandoned / purchaseFailed → ordersPaid, per IST day, with `conversionPct` | New panel. Five stacked client-emitted steps + a server-truth `ordersPaid` line; `conversionPct` is `null` (never `0`) on a day with no views, and **can exceed 100%** — do not clamp it | [SME_INSIGHTS_API.md](./SME_INSIGHTS_API.md) §5 · [SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) §4c |
| **Notifications — broadcast / segment** | `POST /sme/notifications/broadcast` takes `exam` and sends to the FCM topic `exam_<slug>` instead of `all_users`; the resolved `topic` is echoed. `POST …/segment` takes `exam`, now **validated** (an unknown slug used to report `matchedUsers: 0`, indistinguishable from an empty cohort) and echoed back as `exam`. **The tier-predicate regression is fixed (§2.1)** | Add an exam selector to both composers; show the audience you actually addressed from the echoed `topic` / `exam`. ⚠️ `exam: "upsc-cse"` also reaches everyone who has never used the switcher | [SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) |
| **Notification rules** | **Unchanged.** The engine already had its own exam handling | Nothing | [SME_NOTIFICATION_RULES_API.md](./SME_NOTIFICATION_RULES_API.md) · [SME_NOTIFICATION_RULES_PAGE_GUIDE.md](./SME_NOTIFICATION_RULES_PAGE_GUIDE.md) |
| **Surveys** | `targetExams` on create/update — **empty means every exam**, and it is now actually honoured when resolving the audience. `"*"` is rejected (`targetExams does not accept "*" — leave the array empty to target every exam.`). `breakdown=exam` on results, plus `exam` on each raw response row and a trailing `exam` column on the CSV (**appended last**, so no existing column shifts) | Add exam targeting to the authoring form; add the exam breakdown. Label the breakdown **retro-attributed** — a response stores no exam, so it reports the respondent's exam *today* | [SME_FEEDBACK_API.md](./SME_FEEDBACK_API.md) |
| **Feedback** | `feedback_reports.exam_id` is stamped at submit from the request's exam — **forward only, not backfilled**. `GET /sme/feedback/reports?exam=<slug>` narrows to reports filed *from* that exam; `?exam=none` lists the pre-deploy history. Rows carry `examId`; resolved `context` additionally carries the **content's** `examIds`, a different axis that can legitimately differ | Add the exam filter and a "no exam recorded" bucket; do not present `examId` and `context.examIds` as the same thing (a `*`-tagged reel reported by an APPSC user is both) | [SME_FEEDBACK_API.md](./SME_FEEDBACK_API.md) |
| **Filter config** | **Unchanged.** `?exam=` was already there and defaults to `upsc-cse` | Nothing | [SME_FILTER_CONFIG_API.md](./SME_FILTER_CONFIG_API.md) |
| **Exams catalogue** | Unchanged as a contract. It is the source of the slug list every new selector needs, and `GET /sme/exams/:id/content-counts` is the check that catches a mis-tagged load | Populate every exam selector from `GET /sme/exams`. See §2.4 for the `/cms/*` normalisation fix that used to make content invisible in its own catalogue | [SME_EXAMS_API.md](./SME_EXAMS_API.md) |
| **Content ingest** | **New `/sme/content/*`** — the same PYQ / Mains / Simulation writes as `/cms/*`, behind `x-api-key`, with `examIds` **required** on every create and every write audited as `CONTENT_INGEST`. 13 routes (§5). `/cms/*` stays working, un-keyed and legacy: `examIds` there is still optional and still defaults to `upsc-cse` | Point new ingest at `/sme/content/*`. Existing `/cms/*` scripts keep working unchanged — but a forgotten `examIds` there is still a silent `201` into the UPSC catalogue | [SME_CONTENT_INGEST_API.md](./SME_CONTENT_INGEST_API.md) · [CMS_ADMIN_API.md](./CMS_ADMIN_API.md) |
| **Content docs** | `/content-doc-admin` now documents `examIds` (+ the `*` sentinel), takes `?exam=` on the list, and gets the same normalisation + unknown-slug WARN as `/cms/*`. **No keyed mirror exists for this surface** | Tag documents on create; filter the list by exam | [CONTENT_DOC_SME_API.md](./CONTENT_DOC_SME_API.md) |
| **Media** | **Unchanged.** `POST /sme/media/upload-url` has no exam dimension — an image is an image | Nothing | covered inside [SME_OFFERS_API.md](./SME_OFFERS_API.md) / [SME_BANNERS_API.md](./SME_BANNERS_API.md) |

---

## 5. New endpoints

### `GET /sme/analytics/purchase-funnel`

```bash
curl -s -H "x-api-key: $API_KEY_SECRET" \
  "https://app.stanzasoft.ai/api/v1/sme/analytics/purchase-funnel?days=30&exam=appsc-group-1"
```

Returns `{ exam, days, series[], totals, caveats[] }` — raw, no envelope. Each `series[]` day
is `{ day, paywallViewed, upgradeTapped, checkoutOpened, checkoutAbandoned, purchaseFailed,
ordersPaid, conversionPct }`, newest IST day first, every day zero-filled. `days` is 1–90
(default 30; `auth_events` is purged at 90). A **verbatim staging capture** of this call
(2026-09-21, backend `f6329e6`) is in
[SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) §4c, including the full text of all
seven `caveats` strings — note that a zero day's `conversionPct` is `null`, never `0`,
in `totals` as well as in `series[]`.

### `POST /sme/content/pyq`

```bash
curl -s -X POST -H "x-api-key: $API_KEY_SECRET" -H 'content-type: application/json' \
  -d '{"examIds":["appsc-group-1"],"examName":"APPSC Group 1","year":2024,"subject":"Polity",
       "topic":"State Legislature","difficulty":"MEDIUM","source":"PYQ_THEME","nature":"CORE",
       "question":"…","correctAnswer":"B","totalMarks":2,"duration":72,
       "tags":["polity"],"keywords":["legislative council"],"relatedTopics":["State Legislature"]}' \
  "https://app.stanzasoft.ai/api/v1/sme/content/pyq"
```

> 🔴 **`tags`, `keywords` and `relatedTopics` are required and must be non-empty** —
> corrected 2026-09-21 after the earlier version of this body returned a 400 on staging.
> Full capture in [SME_CONTENT_INGEST_API.md](./SME_CONTENT_INGEST_API.md) §3.1.

`201` with the created row, raw. **Read `examIds` back off the response** — it is the
normalised, stored value, not what you sent.

The other twelve `/sme/content/*` routes, all on the same key and the same audit:

| | |
|---|---|
| `POST /sme/content/pyq/bulk` | 1–100 items, `examIds` per item, **all-or-nothing** — the batch is refused before any write |
| `PATCH /sme/content/pyq/:id` | merge; `examIds` rewritten only when the key is present, and `[]` is a `400` |
| `GET /sme/content/pyq` | same filters as `GET /cms/pyq`, plus `?exam=` |
| `POST /sme/content/mains` · `POST /sme/content/mains/bulk` · `PATCH /sme/content/mains/:id` · `GET /sme/content/mains` | the same four, for Mains |
| `POST /sme/content/simulations` | `examIds` required |
| `PATCH /sme/content/simulations/:id` | present `examIds` validated |
| `GET /sme/content/simulations` | includes inactive; plus `?exam=` |
| `POST /sme/content/simulations/questions/bulk` | no `examIds` — questions inherit the test's |
| `POST /sme/content/simulations/:id/assign-questions` | no `examIds`; audit only |

Not mirrored, deliberately: `DELETE`, `GET`-by-id, `GET /cms/simulations/questions`, and
psychometric (which has no exam dimension to get wrong). Keep using `/cms/*` for those.

### New parameters on existing routes

- `?exam=<slug>` — `GET /sme/users`, `GET /sme/orders`, `GET /sme/offers`, `GET /sme/banners`,
  `GET /sme/users/:id/incidents`, `GET /sme/feedback/reports`, `GET /sme/content/{pyq,mains,simulations}`,
  `GET /cms/{pyq,mains,simulations}`, `GET /content-doc-admin`, and all 17 `/sme/analytics/*`.
- `?exam=none` — `GET /sme/orders` (the `UNRESOLVED_EXAM` bucket) and `GET /sme/feedback/reports`
  (the pre-deploy history).
- `?exam=all` — `DELETE /sme/users/:id/premium` only. It is the explicit opt-in to the old behaviour.
- `examId` (body) — `POST /sme/offers` (**required**), `PATCH /sme/offers/:id` (tri-state),
  `POST /sme/users/:id/premium`.
- `exam` (body) — `POST /sme/notifications/broadcast`, `POST /sme/notifications/segment`.
- `examIds` (body) — every `/sme/content/*` create (**required**), every `/cms/*` create and
  `PATCH` (optional, defaults to `upsc-cse`), `/content-doc-admin` create/update.
- `targetExams` (body) — `POST` / `PATCH /sme/surveys/:id`. Empty = every exam; `"*"` rejected.
- `breakdown=exam` — `GET /sme/surveys/:id/results`.

---

## 6. Telemetry now live (app 2.0)

App 2.0 shipped to both stores on **2026-09-17**. The client-event sink (still named
`auth_events`, now a general client-event sink) accepts these types, and 2.0 is the first
build that emits the product ones:

| Group | Types | Filled from |
|---|---|---|
| Auth lifecycle | `login_attempt` · `login_success` · `login_failed` · `otp_verify_failed` | **app 2.0 onward, client-only.** A failure on an older build is recorded nowhere |
| Auth lifecycle | `otp_send_failed` | **server-side** (`OtpService`) *and* the client — backend-observed, so this one is complete |
| Auth lifecycle | `refresh_failed` · `forced_logout` | client |
| Purchase funnel | `paywall_viewed` · `upgrade_tapped` · `checkout_opened` · `checkout_abandoned` · `purchase_failed` | **app 2.0 onward, client-only** |
| Other client signals | `client_error` · `app_opened` · `app_backgrounded` · `chat_conversation_started` | client |

**The funnel** is `paywall_viewed → upgrade_tapped → checkout_opened →
checkout_abandoned / purchase_failed → ordersPaid`. The first five steps are **distinct users
per IST day** from the sink; `ordersPaid` is server truth from `orders` (`status = PAID`,
`is_test` excluded). That asymmetry is the point and it has three consequences the panel must
carry:

1. **Every client step is a FLOOR.** A build that does not emit is invisible, so the whole
   signal is capped at the **adopted share of 2.0** and a rising `conversionPct` can mean
   emitter adoption rather than a better paywall.
2. **`conversionPct` can exceed 100%** — server-truth numerator over a client-floor
   denominator. Do not clamp it.
3. **`conversionPct` is `null`, never `0`, on a day with no views.** A `0%` there reads as
   "we showed the paywall and converted nobody"; the truth is usually that the day predates
   the emitting build.

⚠️ The ingest endpoint is **unauthenticated**, so every type on that list is forgeable. Rows
the backend itself wrote are distinguished by a server-controlled `metadata.source` marker,
not by event type.

---

## 7. Known gaps that remain

Reported, not fixed. Each one is a thing the portal must **say on screen**, because none of
them can be worked around client-side.

1. **There is no per-exam history before the deploy.** `api_usage.exam_id` and
   `api_usage_daily.exam_id` are stamped going forward only. `examCoverage.since` /
   `unknownRows` report it honestly; render "since &lt;date&gt;", never a cliff. It ages out on
   its own — do **not** backfill it by guessing `upsc-cse`.
2. **Client events carry no exam.** `auth_events` has no exam column and nothing to join one
   through, so the purchase funnel's five steps borrow the user's **current**
   `active_exam_id`. `ordersPaid` does not share that weakness (`orders.exam_id` is exact), so
   a per-exam `conversionPct` mixes a retro-attributed denominator with an exact numerator.
   Read it as directional; the all-exams number (omit `exam`) is the exact one.
3. **`clientErrors` and the `authEvents` block cannot be exam-scoped.** On
   `/sme/analytics/release-health`, `clientErrors` stays app-wide beside exam-scoped actives,
   so `clientErrorsPerActiveUser` is an **overstatement** under `?exam=`; the response says so
   in `meta.clientErrorsExamScoped: false`. On `/sme/analytics/onboarding-funnel` the whole
   `authEvents` panel stays app-wide — a pre-auth event has no user to join an exam through.
4. **`GET /sme/offers/:id/abandoned` — the `TAP` signal is not exam-filtered.** It reads iOS
   `upgrade_tapped` / `checkout_opened` events, which carry neither an offer code nor an exam.
   The `ORDER` and `VIEW` signals are campaign-exact; `TAP` is inferred, as it always was.
   (The `campaign` block on both `/abandoned` and `/purchased` *does* carry `examId` — a
   campaign discounts exactly one exam — but the `TAP` rows underneath it are not narrowed
   by it.)
5. **`context.examIds` is `undefined` for simulation questions** on a feedback report.
   `simulation_questions` has no `examIds` column — the container carries the exam. The key is
   omitted rather than sent as `[]`, so "untagged" and "not applicable" stay distinguishable;
   the report's own `examId` is unaffected.
6. **The app does not send `metadata.exam` on client events yet.** The fix is a nullable
   `exam_id` on the sink stamped from the `X-Exam` header at ingest, and it would only work
   forward — the same deploy-day floor as gap 1.
7. **`/cms/*` has no retirement date.** It is not deprecated and is not going away: every
   import script that exists calls it, and making `examIds` required there would break all of
   them at once. New integrations start on `/sme/content/*`; the old surface is simply not
   extended.
8. **`GET /sme/content/question-quality` does not read the exam dimension.** The columns exist
   but that route is on a different controller and was untouched by this release, so a
   mixed-exam pool is still reported as one bank.

`SME_INSIGHTS_API.md` §7 carries the full gap table, including the schema changes each one
would need.

---

## 8. Migrations & deploy order

For our side only — the portal needs nothing from this section except the deploy date.

Two migrations, both additive and idempotent, applied **in order** and then deployed:
`20260921000000_sme_exam_dimension` (the four columns) and `20260921000001_feedback_exam_idx`.

🔴 Migrate and deploy in **one window**: migration 1 regrains the `api_usage_daily` unique key
to include `exam_id`, and the old image's rollup upserts on the key that no longer exists.

Full procedure, the `information_schema` verification (not `_prisma_migrations`) and the
post-deploy smokes: [rollout-and-verify.md](./rollout-and-verify.md) **§7–§8**.

---

## 9. How to confirm the deploy landed

Same pattern as the July note — a **keyless** request. **`401` means deployed and guarded;
`404` means not deployed.** Both routes are new in this release, so on the old image they do
not exist:

```bash
for p in sme/content/pyq sme/analytics/purchase-funnel; do
  printf '%s -> %s\n' "$p" \
    "$(curl -s -o /dev/null -w '%{http_code}' https://app.stanzasoft.ai/api/v1/$p)"
done
# 401 401  → deployed
# 404 404  → not deployed yet
```

Then confirm the two breaking calls actually refuse, **with a key**, by their exact messages —
[rollout-and-verify.md](./rollout-and-verify.md) §7.1. 🔴 If either returns `2xx`, you are on
the old image: a bare revoke there still destroys every exam's entitlement.

**Verified on staging 2026-09-21: keyless probes 401 on 15/15; back-compat capture
identical with and without `X-Exam: upsc-cse`; only additive delta vs pre-deploy is
`topics[+exam_<slug>]`.**

---

## 10. Migration plan: `/cms/*` → `/sme/content/*`

`/cms/*` is public, un-keyed and un-audited, and its `examIds` is optional — which is exactly
the failure mode this exists to close: **an APPSC upload that forgets `examIds` succeeds,
returns `201`, and lands every row in the UPSC catalogue.** Nothing errors.

Both surfaces call the same `CmsPyqService` / `CmsMainsService` / `CmsSimulationService`
writes, so they cannot drift.

| `/cms/*` | `/sme/content/*` | difference |
|---|---|---|
| `POST /cms/pyq` | `POST /sme/content/pyq` | `examIds` required + validated; audited |
| `POST /cms/pyq/bulk` | `POST /sme/content/pyq/bulk` | as above, per item; whole batch refused before any write |
| `PATCH /cms/pyq/:id` | `PATCH /sme/content/pyq/:id` | present `examIds` must be non-empty + known |
| `GET /cms/pyq` | `GET /sme/content/pyq` | identical filters incl. `?exam=` |
| *(same four for mains)* | | |
| `POST /cms/simulations` | `POST /sme/content/simulations` | `examIds` required |
| `PATCH /cms/simulations/:id` | `PATCH /sme/content/simulations/:id` | present `examIds` validated |
| `GET /cms/simulations` | `GET /sme/content/simulations` | identical |
| `POST /cms/simulations/questions/bulk` | `POST /sme/content/simulations/questions/bulk` | audit only (no `examIds`: questions inherit the parent's) |
| `POST /cms/simulations/:id/assign-questions` | same suffix under `/sme/content` | audit only |
| `DELETE /cms/*/:id`, `GET /cms/{pyq,mains}/:id`, `GET /cms/simulations/questions` | not mirrored | keep using `/cms` |

**The one rule: on `/sme/content/*` a missing `examIds` is a `400` by design.** Missing,
`null` or `[]` all return

```
examIds is required: list the exam slugs this content belongs to, or ["*"] for every exam.
```

and a well-formed slug that names no exam returns
`Unknown exam "appsc-grp-1". Create it via POST /sme/exams first.` On `/cms/*` the same input
is a `201` with a server-side WARN nobody reads. `message` is a single string on both — the
check is raised by the controller, not by class-validator, precisely so the portal can match
on one sentence.

After any bulk load, check `GET /sme/exams/:id/content-counts` and read the **`exclusive`**
column. `exclusive: 0` on a module you just loaded into is the signature of a forgotten
`examIds`.

---

## 11. File manifest

Everything in `docs/sme/`. All twenty-six files were last changed **2026-09-21**.

| File | Status |
|---|---|
| [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md) | exam-aware ✅ (new) — this note |
| [README.md](./README.md) | exam-aware ✅ — the index, with a per-doc exam-status column |
| [SME_PORTAL_API.md](./SME_PORTAL_API.md) | exam-aware ✅ |
| [SME_TRIAL_EXTENSION_API.md](./SME_TRIAL_EXTENSION_API.md) | exam-aware ✅ |
| [SME_FILTER_CONFIG_API.md](./SME_FILTER_CONFIG_API.md) | exam-aware ✅ |
| [SME_OFFERS_API.md](./SME_OFFERS_API.md) | exam-aware ✅ |
| [SME_CAMPAIGN_ABANDONED_API.md](./SME_CAMPAIGN_ABANDONED_API.md) | exam-aware ✅ |
| [SME_CAMPAIGN_FORM_GUIDE.md](./SME_CAMPAIGN_FORM_GUIDE.md) | exam-aware ✅ |
| [SME_CAMPAIGN_ANALYTICS_PAGE_GUIDE.md](./SME_CAMPAIGN_ANALYTICS_PAGE_GUIDE.md) | exam-aware ✅ |
| [SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) | exam-aware ✅ |
| [SME_NOTIFICATION_RULES_API.md](./SME_NOTIFICATION_RULES_API.md) | exam-aware ✅ |
| [SME_NOTIFICATION_RULES_PAGE_GUIDE.md](./SME_NOTIFICATION_RULES_PAGE_GUIDE.md) | exam-aware ✅ |
| [SME_FEEDBACK_API.md](./SME_FEEDBACK_API.md) | exam-aware ✅ |
| [SME_APPLE_IAP_API.md](./SME_APPLE_IAP_API.md) | exam-aware ✅ |
| [SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) | exam-aware ✅ |
| [SME_INSIGHTS_API.md](./SME_INSIGHTS_API.md) | exam-aware ✅ |
| [SME_METRICS_DECODER.md](./SME_METRICS_DECODER.md) | exam-aware ✅ |
| [SME_ANALYTICS_FRONTEND_GUIDE.md](./SME_ANALYTICS_FRONTEND_GUIDE.md) | exam-aware ✅ |
| [SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) | exam-aware ✅ |
| [SME_USER_DETAIL_PAGE_GUIDE.md](./SME_USER_DETAIL_PAGE_GUIDE.md) | exam-aware ✅ |
| [SME_EXAMS_API.md](./SME_EXAMS_API.md) | exam-aware ✅ |
| [SME_BANNERS_API.md](./SME_BANNERS_API.md) | exam-aware ✅ |
| [SME_CONTENT_INGEST_API.md](./SME_CONTENT_INGEST_API.md) | exam-aware ✅ (new) |
| [CMS_ADMIN_API.md](./CMS_ADMIN_API.md) | legacy ⚠️ — kept working, not extended; `examIds` optional, WARN not `400` |
| [CONTENT_DOC_SME_API.md](./CONTENT_DOC_SME_API.md) | exam-aware ✅ — but there is **no keyed mirror** for this surface |
| [rollout-and-verify.md](./rollout-and-verify.md) | exam-aware ✅ — §7–§8 are this release |

Not listed: `archive/` (the superseded July 2026 cover note — read it for history, never for
current figures) and `.assets/` (print stylesheet).
