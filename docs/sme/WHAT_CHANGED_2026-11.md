# SME Portal — what changed, 2026-11

Source of truth for the November 2026 backend contract lanes (audit:
[AUDIT_2026-11.md](./AUDIT_2026-11.md) §6–§7). One section per lane. Portal lanes consume
these sections — **P1 (money screens) reads §B1**.

> **Status: committed on `main`, NOT deployed.** Nothing below is live on staging or
> production until the owner deploys it. Every JSON example here is **"from DTO — replace
> after staging deploy"**: built from the DTOs and the specs, with values taken from the
> real staging runs logged during the lane, not captured from an HTTP response.

---

## §B1 — Money truth contract (backend lane B1, 2026-11-03)

Commits on `main` (in order): B1.1 `e383123` · B1.2 `5890cbc` · B1.3 `fa45745` ·
B1.4 `67d1bda` · B1.5 `2f6e310` · B1.6 `26c145e` · B1.7 (this doc).
No migration, no new table — every new field is computed from existing tables.

### B1.0 Rules that apply to everything below

1. **Every new field is ALWAYS present.** Nullable ones are `null`, never omitted (the
   portal's typed client and kotlinx both treat a missing key as an error).
2. **Every new query parameter is declared** on its DTO. The API runs
   `forbidNonWhitelisted`, so an undeclared parameter is a `400` — send only what is
   listed here.
3. **Errors** keep the usual shape `{ success: false, message }`; bodies are raw (no
   envelope). A bad date, an unknown exam, `from` without `to` → `400`. Nothing falls
   back to an empty list on failure.
4. **Internal accounts are excluded by default** (owner decision 2). One rule,
   `src/modules/sme/services/sme-internal-accounts.util.ts`. A user is internal when ANY of:
   - email ends in `@stanzasoft.com`, `@stanzasoft.ai`, `@prepmonkey.com`, `@prepmonkey.ai`
     or `@designasylum.in` (case-insensitive, whole domain);
   - phone is a store-review demo login (`+919898989898` App Store, `+919876543210` Play —
     the OTP bypass numbers in `otp.service.ts`; on staging one of them is a gmail account,
     which is why the domain rule alone is not enough);
   - they own an App Store **sandbox** order (`orders.is_test`).

   `includeInternal=1` (or `true`) opts back in. Rows carry `isInternal`; aggregates echo
   `includeInternal`. **A single user's own page (`/sme/users/:id`, `/snapshot`) is never
   filtered**, only labelled. Team members on personal Gmail / Apple-relay addresses are
   not caught by any rule (known floor).
5. **Comps and sandbox are labelled, never money** (owner decision 3). `payment_source =
   MANUAL` (a staff grant: PAID, no money moved; its amount is the notional list price) →
   `isComp: true`; App Store sandbox → `isSandbox: true`. Rows stay listed; every money
   total excludes them.
6. **Dates are IST calendar days** (bug M4). A bare `YYYY-MM-DD` start = that day's
   00:00 IST inclusive; a bare end = **the whole day** (`<` next 00:00 IST), so
   `startDate = endDate` is one full day. A full ISO timestamp is taken literally.
7. **No account-wide "premium" figure** (owner ruling 2026-11-03). Every paid / comped /
   revenue figure is **per exam**: the headline is the exam asked for, or **`upsc-cse`**
   (the legacy default) labelled `examDefaulted: true`; `byExam[]` lists every exam and
   `allExams` sums them. Account-level state (`isPremium`, `premiumState`,
   `premiumExpiresAt`, `status=SUBSCRIBED`, trial) must be read as the **UPSC-CSE exam's**
   state — the same rule as `sme-exam-scope.util.ts` (the default exam also matches users
   with no active exam). Never render a bare "premium" count.
8. **Per-exam rows are the truth; the NULL shim lives in one place** (owner rule). A
   coming data backfill will set `user_profiles.active_exam_id = 'upsc-cse'` for onboarded
   users with NULL and create UPSC `user_exam_entitlements` rows for legacy-premium users.
   This lane's code is correct before and after it: person-axis exam scoping goes only
   through `sme-exam-scope.util.ts` (`examUserSql` / `examUserWhere`), order-axis scoping
   uses `orders.exam_id` exactly, and no new code derives paid/trial state from
   `premium_expires_at`, `trial_ends_at` or `status = SUBSCRIBED` (the campaign chase
   list's "already converted" check moved FROM `premium_expires_at` TO a live
   `user_exam_entitlements` row for the campaign's exam). After the backfill the numbers
   shift only where the backfill itself changes the data (more `upsc-cse` people found by
   the literal; more `holdsExam` exclusions on UPSC campaigns) — no code change needed.

### B1.1 `includeInternal` — where it applies

| Route | Default | Row flag / echo |
|---|---|---|
| `GET /sme/transactions` | internal buyers' payments excluded (page + `total`) | row `isInternal` |
| `GET /sme/orders` | excluded | row `isInternal` (nested `payments[]` too) |
| `GET /sme/webhooks` | events whose user is resolvable AND internal are excluded; user-less events kept (unknown ≠ internal) | row `isInternal` |
| `GET /sme/payment-attempts` | excluded | row `isInternal`, echo `includeInternal` |
| `GET /sme/users` | **excluded from the list and `total`** | row `isInternal` |
| `GET /sme/analytics/summary` | DAU/WAU/MAU + `money` exclude internal | echo `includeInternal` |
| `GET /sme/analytics/purchase-funnel` (+ `/users`) | every step and the paid side | echo `includeInternal` (in `window` on `/users`) |
| `GET /sme/offers/:id/abandoned` | rows, `total`, `funnel` | row `isInternal`, echo `includeInternal`, `excluded.internal` |
| `GET /sme/offers/:id/purchased` | rows | row `isInternal`, echo `includeInternal` |
| `GET /sme/users/:id`, `/sme/users/:id/snapshot` | **never filtered** | `isInternal` on the user |

⚠️ `/sme/users` now hides staff by default — a support search for a staff account needs
`includeInternal=1`. The portal's own 2-domain filter (`OfferAnalytics.tsx:92`) can go.

### B1.2 Payment attempts (M1) — `GET /sme/payment-attempts` + `paymentAttempts[]`

```
GET /sme/payment-attempts?page=&limit=&userId=&exam=&kind=&status=&from=&to=&search=&includeInternal=
```

| Query | Meaning |
|---|---|
| `userId` | one user |
| `exam` | `orders.exam_id` / `subscriptions.exam_id`; `none` = no exam; unknown → 400 |
| `kind` | `APPLE` · `RAZORPAY_ONE_TIME` · `UPI_AUTOPAY_MANDATE` · `COMP` |
| `status` | verbatim source status: orders `CREATED/PAID/FAILED/CANCELLED`; mandates `PENDING/ACTIVE/PAST_DUE/PAUSED/CANCELLED/EXPIRED` |
| `from`, `to` | IST calendar days on `createdAt` (rule 6) |
| `search` | user email / name / phone, or the FULL external id (`sub_…`, `order_…`, Apple id) |

One row per attempt, newest first. A **computed union**: every order (Apple purchase /
renewal, Razorpay one-time order, MANUAL comp) + every Razorpay mandate in
`subscriptions`. A mandate's charge orders (receipt `sub_<mandate>`) are **folded into the
mandate row** (`everCharged`, `lastEventAt`), not listed again.

Row (same shape in the list, in `GET /sme/users/:id` → `paymentAttempts[]` and in
`GET /sme/users/:id/snapshot` → `entitlement.paymentAttempts[]`; the last two = newest 20,
never filtered; `recentOrders` is unchanged for compatibility):

```json
{
  "id": "e9fd118e-…",
  "userId": "17ee1dee-…",
  "kind": "UPI_AUTOPAY_MANDATE",
  "status": "PENDING",
  "amountPaise": 590000,
  "currency": "INR",
  "examId": "upsc-cse",
  "planType": "ANNUAL",
  "createdAt": "2026-10-31T16:47:22.482Z",
  "lastEventAt": "2026-10-31T16:47:22.482Z",
  "ref": "sub_TuaQ…",
  "source": "subscriptions",
  "isInternal": true,
  "isSandbox": false,
  "everCharged": false,
  "user": { "email": "…", "name": "…", "phoneNumber": null }
}
```
*(from DTO — replace after staging deploy; values from the staging run of the service.)*

Page: `{ data, total, page, limit, hasMore, includeInternal, note }`.

| Field | Notes |
|---|---|
| `amountPaise` | orders: `orders.amount` (INR; for `COMP` the **notional** list price — no money moved). Mandates: expected first charge (`first_charge_paise`), `null` on old rows |
| `ref` | external id cut to 8 chars + `…`; `null` for a COMP |
| `isSandbox` | `orders.is_test`; always `false` for a mandate |
| `everCharged` | **additive beyond the owner's field list** — order PAID (not a comp) or mandate `first_charge_granted_at` set. Without it a CANCELLED mandate that billed for a year and one never approved look the same. A PENDING mandate with `everCharged: false` = "tried, never charged" |
| `user` | **additive beyond the owner's field list** — `{ email, name, phoneNumber }` so the global list can say who without N lookups; all null when the user row is gone (subscriptions keep no FK) |

**Render:** "Money" block on the profile = `paymentAttempts[]` (kind · status · date ·
amount, COMP labelled). On staging the PENDING-mandate user `17ee1dee` returns a
`UPI_AUTOPAY_MANDATE / PENDING / everCharged=false` row; production user `3a5a9b8f` will
return its 3 mandates once deployed.

### B1.3 Comps, sandbox and money totals (M2, M3, M12)

**Row flags (always present):**

| Route | New fields |
|---|---|
| `/sme/transactions` rows | `isInternal`, `isComp` (parent order MANUAL), `isSandbox` (parent order `is_test`) |
| `/sme/orders` rows (+ nested payments) | `isInternal`, `isComp`, `isSandbox` (= `isTest`, kept) |
| `/sme/webhooks` rows | `isInternal`, `isComp` (always `false` — a comp emits no provider event), `isSandbox` (Apple: decoded from the stored JWS `data.environment` — catches the 6 pre-gate sandbox rows stored `processed`; Razorpay: `false`), `isDowntime` (B1.4) |

`?includeTest=false` now really means false (it used to be read as `true` under implicit
conversion).

**`GET /sme/analytics/summary` → new `money` block.** The portal's `/analytics` and `/`
money cards were client-side sums of our list pages (S01). Switch them to this:

```json
"money": {
  "window": "all-time",
  "exam": "upsc-cse",
  "examDefaulted": true,
  "revenuePaise": 590000,
  "paidOrders": 1,
  "payingPeople": 1,
  "bySource": { "apple": { "revenuePaise": 0, "paidOrders": 0 }, "razorpay": { "revenuePaise": 590000, "paidOrders": 1 } },
  "nonInrOrders": 0,
  "comps": { "orders": 0, "people": 0, "notionalPaise": 0 },
  "sandboxOrders": 0,
  "byExam": [ { "examId": "upsc-cse", "revenuePaise": 590000, "paidOrders": 1, "payingPeople": 1, "bySource": {…}, "nonInrOrders": 0, "comps": {…}, "sandboxOrders": 0 } ],
  "allExams": { "revenuePaise": 590000, "paidOrders": 1, "payingPeople": 1, "bySource": {…}, "nonInrOrders": 0, "comps": {…}, "sandboxOrders": 0 },
  "appleEvents": { "production": 0, "sandbox": 0, "unknown": 0, "scope": "all-exams" },
  "note": "All-time, PER EXAM …"
}
```
*(from DTO — replace after staging deploy; staging default values.)*

| Portal card today | Switch to |
|---|---|
| "Revenue (captured)" | `money.revenuePaise` (headline exam) / `money.allExams.revenuePaise`; ₹ = paise / 100 |
| "N payments" | `money.paidOrders` |
| "Revenue by source" | `money.bySource` (no MANUAL bar; show `money.comps.notionalPaise` separately, labelled "comped, not received") |
| "Revenue by exam" | `money.byExam[]` |
| "Paying Subscribers" / "Premium" | `money.payingPeople` per exam — never a bare premium count |
| "Apple events recorded" | `money.appleEvents.production` (sandbox shown separately) |
| "% success" | drop the tile (M6); failures come from `payment-attempts` |

Rules: real money = PAID orders minus MANUAL and sandbox; revenue = CAPTURED payments on
them, INR only (non-INR Apple storefronts counted in `nonInrOrders`, never summed), gross
of refunds; internal excluded unless `includeInternal=1`; `appleEvents` is cross-exam
(a notification has no exam). `summary` without `exam` is still the DAU/WAU/MAU headline;
it now also echoes `includeInternal` (and is no longer byte-identical to the pre-November
body — owner decision).

**Purchase funnel (`GET /sme/analytics/purchase-funnel`)** — see
[SME_INSIGHTS_API.md §5](./SME_INSIGHTS_API.md):

| Field | Change |
|---|---|
| `exam` | **always a slug** now; omitted → `upsc-cse` |
| `examDefaulted` | new, `true` when `exam` was not passed |
| `ordersPaid` (series + totals) | **FIRST real payments only**: no comps, no renewals, no sandbox, no internal |
| `renewals`, `comps` (series + totals) | new — Apple renewals / UPI AutoPay cycle ≥ 2; staff grants |
| `people` | new — `{ paywallViewed, upgradeTapped, checkoutOpened, checkoutAbandoned, purchaseFailed, paid, conversionPct }`, distinct people over the whole window = the length of the per-step list |
| `peopleNote` | new — why `people` < `totals` (per-day sums) |
| `byExam[]`, `allExams` | new — paid side `{ ordersPaid, renewals, comps, paidPeople }` per exam / across exams |
| `window` | new — `{ from, to }`, `to` exclusive |
| `includeInternal` | new echo |

Tiles should show `people.*`; the conversion headline `people.conversionPct`.
"Purchase failed" tooltip: "our server refused to start the purchase" (not "the payment
itself failed"); "Checkout abandoned": "left the payment sheet (no reason captured)".

**Campaign analytics (`/sme/offers/:id/abandoned`, `/purchased`)** — one campaign is one
exam (`campaign.examId`), so its money is per-exam by construction:
- comps (MANUAL) never count as joined / paid / revenue / unclaimed discount, and never
  appear on `/purchased`;
- new `excluded: { internal, paidAfterwards, compedByStaff, holdsExam }` — everyone who
  reached for the campaign but is not on the chase list, one reason each (precedence in
  that order). **`compedByStaff` replaces the "already converted" label**; the check is
  now per the campaign's exam (`orders.exam_id` / live `user_exam_entitlements`), not the
  person-level `premium_expires_at`.

### B1.4 Dates and webhooks (M4, M5)

- `startDate` / `endDate` on `/sme/transactions`, `/sme/orders`, `/sme/webhooks` follow
  rule 6. Staging proof: `startDate=endDate=2026-10-12` → 15 payments = the IST day.
- New provider events are linked to `userId` / `orderId` **at ingest**, after handling,
  only where the payload names something we know (Razorpay payment / order / refund /
  mandate id / notes.userId; Apple transaction / original transaction). Handling is
  unchanged. Old rows: `scripts/backfill-payment-event-links.ts` (dry run by default,
  `--write` fills NULLs only, idempotent; staging done — 33 of 38 rows linked; **production
  is the owner's**: `--prod --write`). Once linked, `examId` on webhook rows and the
  profile's "Payment webhooks" count become real.
- `GET /sme/webhooks?kind=money|downtime|all` (default `all`): `money` hides the Razorpay
  `payment.downtime.*` notices (~99.5% of prod rows); rows carry `isDowntime`. Render
  downtime as one provider-health line, not as rows.

### B1.5 Apple list price per exam (M7)

New Apple orders store `listAmount` = the bought exam's `exam_plans.price_in_paise` for
that plan type (APPSC ₹499, not UPSC's ₹599); the global `app_config` price is only the
fallback when the exam has no plan row (logged). **Existing rows are not rewritten** (no
data fix in this lane): the 6 prod APPSC Apple orders keep `listAmount` 59900 and still
draw the fake strike until the owner decides on a one-off correction. Staging cannot show
the difference — its APPSC plans are priced like UPSC (59900 / 590000).

### B1.6 Funnel per-step people list (D22)

```
GET /sme/analytics/purchase-funnel/users?step=paywall_viewed|upgrade_tapped|checkout_opened|paid|abandoned|failed&days=&from=&to=&exam=&includeInternal=&page=&limit=
```
`step` required; `limit` default 50, max 200; `from`/`to` together, IST days, ≤ 90 days.

```json
{
  "step": "failed",
  "window": { "from": "…", "to": "…", "days": 90, "exam": "upsc-cse", "examDefaulted": true, "includeInternal": true },
  "data": [ { "userId": "17ee1dee-…", "displayName": "…", "kind": "UPI_AUTOPAY_MANDATE", "at": "2026-10-31T11:53:18.913Z", "examId": "upsc-cse", "isInternal": true } ],
  "total": 2, "page": 1, "limit": 50, "hasMore": false,
  "note": "One row per person: …"
}
```
*(from DTO — replace after staging deploy; values from the staging run.)*

`total` = the funnel's `people[step]` for the same query (staging: 24/24 step × scope
checks equal). `kind`: `APPLE`, `RAZORPAY_ONE_TIME`, `UPI_AUTOPAY_MANDATE`, `RAZORPAY`
(Razorpay, rail not recorded) or `null` (nothing picked — every paywall view). Each row
links to `/app-users/:userId`.

### B1.7 What the portal lane P1 must do (checklist)

1. Profile: render `paymentAttempts[]` as the Money block; drop the "Payment webhooks" tile
   until the prod backfill has run.
2. `/transactions`: send IST dates as bare `YYYY-MM-DD` (already does); add an Attempts tab
   on `/sme/payment-attempts`; label `isComp` / `isSandbox` / `isInternal` rows; webhooks
   default `kind=money` + a downtime line from `kind=downtime&limit=1` → `total`.
3. `/analytics` + `/`: money cards from `summary.money` (per exam; headline says which exam;
   no bare "premium"); delete the client-side sums.
4. Purchase funnel: tiles from `people`, drill-down from `/purchase-funnel/users`, exam
   label from `exam` + `examDefaulted`, custom range via `from`/`to`.
5. Campaign page: `excluded.*` for the bridge; "already converted" → "comped by staff".
6. Remove the portal's local internal-domain filter; pass `includeInternal=1` only behind
   an explicit "include staff" toggle.

---

## §B4 — Keyless parity (backend lane B4, 2026-11-03)

> Status: committed on branch `lane/b4` (not yet on `main`), NOT deployed. Shapes are from code + jest; only the edit-log SQL was run (read-only) on staging.

Owner ruling 2026-11-03: every keyless route the portal calls (`/cms/*`, `/content-doc-admin`,
`/reels`, `/videos/*`) gets a **keyed `/sme/*` twin with its functionality replicated end to
end**, and `/cms/*` is **not** removed. Proven by `src/modules/sme/sme-keyless-parity.spec.ts`:
one fixture, every keyless controller and every keyed twin, the real services — 40 read routes
return the same status, rows and error body on both sides; the deliberate differences are
pinned in the same file. No migration, no new table. Issues closed: K1, K2, K3 (backend half),
K4 (backend half), K6 (backend half).

### B4.1 — Delete is soft + attributed; GET by id (K1)
- New keyed `GET /sme/content/{pyq,mains,simulations}/:id`, `GET /sme/content/simulations/questions?externalIds=`.
- New keyed `DELETE /sme/content/{pyq,mains,simulations}/:id`: **soft** (`isActive=false`), body
  `{actor, reason, note?, reportId?}` required (one-sentence 400 otherwise), and a
  `CONTENT_DELETE` row in `sme_audit_log` (reason, full before-snapshot, asserted actor)
  written **in the same transaction**. `pyq_attempts` / `mains_evaluations` have no FK, so
  the soft-deleted id stays resolvable (staging today: 42 attempts already orphaned by hard
  deletes). The keyless `/cms/{pyq,mains}/:id` DELETE is unchanged (hard).
- ⚠️ **One behavioural change on a keyless route:** `DELETE /reels` (every reel + every Mux
  asset + every blog by cascade) now needs the body `{"confirm": "DELETE ALL REELS"}` — anything
  else is `400 Refusing to delete every reel: send {"confirm": "DELETE ALL REELS"} in the body.`
  and nothing is touched (checked before the Mux deletions). A confirmed wipe writes
  `REELS_DELETE_ALL`. There is no keyed delete-all. **The portal's "Delete All" button must
  send the phrase or it will 400 after the deploy.**

### B4.2 — Psychometric on `/sme/content/psychometric/*` (K3)
All 11 `/cms/psychometric/*` routes, same suffixes, same service and rows. Differences:
writes audited; DELETEs soft + attributed (keyless: hard); test-set `language` strict; the
test-set list honours `isActive` / `search` / `language` (keyless ignores them). **No
`examIds` and no question `language`** — the columns do not exist and this lane adds no
migration (owner decision if wanted). The learner `GET /psychometric/test-sets/:id/questions`
now skips inactive questions (staging: 2,015/2,015 active — no visible change today).

### B4.3 — Content-doc re-sync keeps learners' answers; `/sme/content/docs` (K2, K3)
- **Both surfaces:** `PATCH` with changed `rawMarkdown` and `POST :id/reparse` no longer
  `deleteMany` the document's questions (`user_question_attempts` cascade on question delete —
  every edited portal re-sync wiped answers). Questions are upserted in place: matched by text
  + options first, then by position; moved/unwanted rows parked first (the unique key
  `(document_id, section_order, question_order)` is non-deferrable); dropped questions become
  `isActive=false`, never deleted. Progress counts (`submitAttempt`, `markAsRead`) consider
  live questions only. The **slug is kept** unless the keyed body sets `slug` (it was
  regenerated from the title + timestamp on every titled PATCH); `status` changes only when
  sent (unchanged — the portal still sends `PUBLISHED` on re-sync; that is a portal fix, P5).
- New keyed `/sme/content/docs`: create (`examIds` required, `language` strict), list (+
  `?language=`), `by-source/:sourceId`, `:id`, PATCH (+ `slug`), `:id/reparse`, DELETE (soft,
  attributed). No `reparse-all`.

### B4.4 — Reels on `/sme/reels`; blogs by reel id (K1, K3, K6)
- New keyed `GET /sme/reels` (≡ `GET /reels`, + `?exam=`, `?language=`, not behind
  `@Feature('reels')`), `GET /sme/reels/:id` (+`hasBlog`), `PUT /sme/reels/bulk` (strict
  examIds/language), `PATCH /sme/reels/:id`, `POST /sme/reels/upload-url`,
  `GET /sme/reels/:id/video-status` (same Mux calls and `{success,data}` shapes as `/videos/*`),
  `DELETE /sme/reels/:id` — soft: `status: "DELETE"` (the existing hidden state), Mux asset and
  blog kept, attributed audit row; restore with `PATCH {status:"READY"}`.
- The learner feed, by-id read and bookmark list now exclude `status = DELETE` reels
  (staging: 0 such reels — no visible change today). Admin lists still show them.
- `GET /sme/blogs?reelId=a,b,c` (≤ 100) returns exactly those reels' blog state.

### B4.5 — Content edit log on our side
`GET /sme/content/edits?from&to&bank&actor&action&page&limit` (IST days, default last 30,
newest first) and `GET /sme/content/edits/:id` (+ before/after) over `sme_audit_log`
content actions. Every row key always present; legacy rows get `bank` derived from their
route. The generated SQL was executed read-only on staging (2,264 rows in 2026; 2,185 PYQ).
Writes other than deletes carry no actor — attributing edits needs an owner decision on the
portal sending one.

### B4.6 — Keyed bulk is per row (K4)
`POST /sme/content/{pyq,mains}/bulk` now validates **each row** — `examIds` (required, known),
`language` (strict), PYQ `correctAnswer` (one uppercase A–E, or null with `boardDeleted`),
`year` (1950 … next year; a blank Excel Year arrives as 0) — writes the rows that pass in one
transaction and returns `{ count, created, rejected, results: [{index, status, id, error}] }`.
⚠️ **Contract change for callers of the keyed bulk** (the portal proxies it but does not use
it yet): it was all-or-nothing returning `{count}`. Every row rejected → `400` with the row
errors as `message`. `GET /sme/content/{pyq,mains}` take `?language=`. `/cms/*/bulk` unchanged.

### What the portal lane (P5) has to do
Switch every `/cms/*`, `/content-doc-admin`, `/reels*`, `/videos/*` call to the twins above;
send `{actor, reason}` on deletes; send `examIds` + `language` per Excel row and read
`results[]`; stop sending `status: "PUBLISHED"` on re-sync; load blogs with `?reelId=`; send
the confirmation phrase on Delete-All (or remove the button).

---

## §B3 — Users-list + control contracts (backend lane B3, 2026-11-03)

Branch `lane/b3` (commits B3.1 → B3.7). Nothing here is deployed yet: every staging proof
below was run **read-only** (the real Prisma builders / service methods against the staging
database, compared with hand SQL); the HTTP-level checks are listed in §B3.8 for after the
owner's staging deploy. No migration, no new table.

### B3.0 Rules that apply to everything below

- **The account-wide word "premium" is retired (owner ruling 2026-11-03).** Access is per
  exam. `premium=` and the new `premiumState=` filter are **LEGACY** (UPSC-era, account-wide);
  `isPremium` / `premiumState` stay on rows for compatibility. Use `access=` + `exam=` and the
  row's `access[]`.
- **Backfill-stable (owner rule 2026-11-03).** The access axis and the push cohorts read
  `user_exam_entitlements` + `active_exam_id` (NULL ≡ upsc-cse **only** via
  `sme-exam-scope.util`'s `examUserWhere`) + the trial clock (`created_at`, the exam's
  `trial_policy`, an SME-set `trial_ends_at`). The `user_auth` mirror (`status=SUBSCRIBED`,
  `premium_expires_at`, `subscription_source`) enters through **one** mapping
  (`sme-user-access.util.ts`, `hasEntitlementWhere` / `withLegacyPremiumRow`): a SUBSCRIBED
  user with **no entitlement row in any exam** reads as holding one virtual `upsc-cse` row
  (`source = COALESCE(subscription_source,'MANUAL')`, `expires_at = premium_expires_at`,
  not revoked) — the 2026-09-08 migration's shape. ⚠️ **The planned backfill must use exactly
  that predicate ("no row in any exam", not "no upsc-cse row") and that row shape**, or the
  answers will move; a SUBSCRIBED user who holds an APPSC row is a per-exam-era buyer and
  must NOT get an invented UPSC row (staging has exactly one such user).
- Internal accounts: the users list keeps B1.1's default exclusion (`externalUserWhere`,
  `includeInternal=1` to opt in); B3's clauses are ANDed next to it.
- Dates: a bare `YYYY-MM-DD` is an IST day, the end bound is the next IST midnight
  (exclusive), so `from=to=D` is exactly day D; an ISO instant is literal. The new B3.6
  routes use B1's `istDateRange` (`sme-date-range.util.ts`).

### B3.1 `/sme/users` — Access axis, ONBOARDING, language, joined dates (L2)

New query params (all declared on `SmeUserListQueryDto`):

| Param | Values | Notes |
|---|---|---|
| `access` | `paid` `comped` `not_onboarded` `trial` `lapsed` `trial_over` | **What the person HOLDS in `exam`** — that exam's entitlement rows (+ the legacy mapping for upsc-cse) and that exam's trial length — over **every** user, whichever exam they have selected (B3.1c: an APPSC buyer in UPSC mode is `exam=appsc-group-1&access=paid`; with `access`, `exam` does not apply the selected-exam clause — the plain list without `access` still does). Partition: every user in exactly one value, per exam. **No `exam` → evaluated against `upsc-cse`** and the body says `examDefaulted: true`. Definitions: SME_PORTAL_API §1.1a. |
| `language` | a code (`hi`, `pt-br`) or `none` | `user_profiles.ui_language`; `none` = never chosen. |
| `createdFrom` / `createdTo` | IST date or ISO instant | Replaces the portal's full scan for "Joined". 400 on `2026-02-31` or inverted. |
| `status=ONBOARDING` | — | Now filterable (5,020 prod users were not). |
| `premium=false` | — | Now the exact complement of `true` (was silently ignored). LEGACY. |
| `premiumState` | `Premium` `Trial` `Trial Ended` `Churned` `Downloaded` | LEGACY (was a 400). Returns exactly the rows that show the label. |

Response: the list body always has `examDefaulted` (boolean); **every row** (list, detail,
PATCH response) has `access[]` = `[{ examId, state, until, source }]`, one line per active
exam (+ any exam the user holds a row in). `PATCH /sme/users/:id` accepts
`status: "ONBOARDING"` **only when the user already is ONBOARDING** (the edit modal resends
it with a name change — that used to 400); moving someone into ONBOARDING is a 400.

Staging proof (read-only, after B3.1c): the Prisma `where` for every value × exam
(upsc-cse, appsc-group-1) equals hand SQL at four instants (now, 2026-06-25, 09-20, 10-03 —
the last three exercise the 14 / 3 / 0-day policies); for each exam the six values sum to
all 15 users; every user's `access[]` line for **every** exam equals the bucket the filter
puts them in (30/30 × 4). Now: upsc-cse paid 1 · comped 1 · not_onboarded 5 · lapsed 1 ·
trial_over 7; appsc-group-1 paid 1 · comped 2 · not_onboarded 6 · lapsed 1 · trial_over 5
— the 2 APPSC comps belong to users NOT in APPSC mode, which the pre-B3.1c filter missed.

### B3.2 Push cohorts use each exam's trial length (L3)

`POST /sme/notifications/segment`: `premium` = holds a live entitlement (paid ∪ comped),
`trial` = access `trial`, `free` = the rest; scope = `exam` when sent, else every user. The
trial window was a fixed 14 days for everyone (prod 1,609 reached vs 274 on trial). **Also
fixed:** `free` silently dropped every ACTIVE user past their trial (NOT over a NULL
`trial_ends_at` comparison) — staging: the old cohorts summed to 11 of 15 users. Staging
proof: the real `resolveSegment` equals the hand classes collapsed, 4 instants × 3 scopes.

### B3.3 Notification settings accept the echoed `priority` (C1)

`PATCH /sme/notification-rules/settings` accepts `priority` and **ignores** it (dropped by
the DTO, never stored; the category order is not settable). The portal's Guardrails Save and
the kill switch — which post the `GET` object back — had never worked (prod
`notification_config` NULL, 0 audit rows). The S07 probe bodies now validate. Max per day
outside 0–5 is a 400 (the Swagger text that said "corrected" is fixed).

### B3.4 Plan edits send changed fields only (C2)

`POST /sme/exams/:id/plans`: `planType` / `priceInPaise` optional on an **edit** (required
to create; `null` = 400); strike-only / price-only / store-id-only edits merge onto the stored
row. Blank `appleProductId` / `razorpayPlanId` = 400 (omit = keep, `null` = clear). All S07
plan-probe bodies validate.

### B3.5 Explicit-null clears + validation parity (C4, C8)

- Rule `audience` PATCH: key `null` or `[]` removes that filter; `audience: null` = everyone.
- Rule `cooldownDays` / `priority` `null` → 400 (was a NOT NULL 500).
- Feedback config `survey.*` / `limits.*`: integer bounds (table in SME_FEEDBACK_API §3);
  `null` and unknown keys 400.
- Surveys: `description: null` and `endAt: null` **already** cleared server-side (now
  spec-proven — the C8 survey half is portal-only); `startAt`/`endAt` ISO-validated.

### B3.6 Windows on control-screen reads (D23)

| Route | New params | Echo |
|---|---|---|
| `GET /sme/feedback/chat/summary` | `from`, `to`, `exam` (slug or `none`, reports rule) — every count scoped | `window`, `exam` |
| `GET /sme/feedback/chat/messages` | `from`, `to`, `exam` | — |
| `GET /sme/surveys/:id/results` | `from`, `to` on `answered_at` | `window` |
| `GET /sme/notification-rules/:id/runs` | `from`, `to` on `ran_at`; `limit` 1–100 (400 outside) | — |
| `GET /sme/notification-rules/:id/analytics` | `from`, `to` (win over `days`); `days` 1–365 | `from`, `to` |

All sources are permanent tables, so no retention label is needed. Staging proof (real
service methods, read-only): rule 27bc6804 `from=to=2026-10-29/30/31` → sent 0/1/0; chat
summary all 6 · 10-15 → 2 · 10-11 → 2 · 10-12..14 → 1 · `exam=upsc-cse` 2 — each equal to
hand SQL.

### B3.7 What the portal lanes P3 / P4 must do (checklist)

1. **P3 `/app-users`**: one Access select → `access=` (+ `exam=`; show "UPSC (default)" when
   `examDefaulted`; label the count "holds/had <exam>", not "studying <exam>" — with
   `access` the list spans every user, not only those who selected the exam); Language → `language=`; Joined → `createdFrom`/`createdTo` (plain
   `YYYY-MM-DD`) — with these the 163-call scan and Export's scan path can go; add
   ONBOARDING to the status list (or drop Status for a "Deactivated" toggle =
   `status=SUSPENDED`); render the row's `access[]` instead of the Premium badge.
2. **P4 Guardrails**: either body works now; prefer sending only the four settable fields.
3. **P4 Plans**: keep sending changed keys only; send `null` (not `""`) to clear a store id.
4. **P4 Rule editor**: send `null` for a cleared audience field; never send `null`
   cooldown/priority — omit them.
5. **P4 Surveys**: send `null` to clear description / end date.
6. **P4 Feedback**: chat cards + survey results + rule runs/analytics → window pills and
   from–to via the params above; chat gets the exam selector (`none` = "no exam recorded").

### B3.8 To verify after the staging deploy (HTTP)

- `PATCH /sme/notification-rules/settings` with the GET body and `enabled` toggled → 200,
  `GET` reflects it, one `NOTIFICATION_SETTINGS_UPDATE` audit row; toggle back.
- `POST /sme/exams/upsc-cse/plans {"planId":"annual","strikePriceInPaise":<stored>}` → 200,
  `priceInPaise` unchanged.
- `GET /sme/users?access=trial&exam=appsc-group-1` total = the §B3.1 SQL; `GET /sme/users?access=paid`
  → `examDefaulted: true`.
- An ONBOARDING user's `PATCH {name, status:"ONBOARDING"}` → 200 (staging has no ONBOARDING
  user today; pick one on prod read-only or create one on staging via a fresh sign-up).

---

## §B6 — Follow-ups surfaced by the portal lanes (backend lane B6, 2026-11-03)

Commits on `main` (in order): B6.1 — none (BLOCKED, see below) · B6.2 `b637832` ·
B6.3 `49703b6` · B6.4 `a890bda` · B6.5 (this doc). **Committed, NOT deployed.** No
migration, no new table. Staging proofs below were run **read-only** (the real service
methods / their rendered SQL against the staging database, compared with hand SQL); the
HTTP checks are in §B6.6 for after the owner's staging deploy.

### B6.1 Usage maintenance jobs — BLOCKED (the stated cause is not in the code)

The plan said the `api_usage` rollup/purge jobs were skipped because
`NotificationSchedulerService.onModuleInit` returns early under
`NOTIFICATION_ENGINE_ENABLED=false`. They are not: `ApiUsageRollupService` registers
both repeatable jobs (`api-usage-rollup` / `api-usage-purge`, 00:30 / 01:30 IST) in its
**own** `onModuleInit`, gated only by **`API_USAGE_ROLLUP_ENABLED`** (default on;
`false` disables). **`.env.staging` sets `API_USAGE_ROLLUP_ENABLED=false` on purpose**
("its purge deletes rows" — docs/staging.md "Traps"). Production does not set it, so prod
runs both.

Staging, 2026-11-03: `api_usage_daily` **0 rows ever**; raw `api_usage` 116,113 rows from
2026-09-09 (the 30-day purge has never run either). No code was changed: renaming the flag
would silently start the row-deleting purge on staging, overriding a deliberate setting.
**Owner decision:** to get rollups on staging, set `API_USAGE_ROLLUP_ENABLED=true` in the
staging task definition (the purge then trims raw `api_usage` to 30 days and `auth_events` to
90 days on staging). If rollup-without-purge is wanted, that is a new flag — ask for it.

### B6.2 `/sme/analytics/users` rows: `daysStudied`, `sessions`, `minutesInApp`

Every row gains three integers over the route's window (the same window as `actions30d`),
**always present**, `0` = nothing:

| field | rule |
|---|---|
| `sessions` | the profile's `timeInApp` sitting rule: runs of `api_usage` + `auth_events` rows with no gap over 30 min |
| `minutesInApp` | Σ (last − first) per sitting — a floor; = the profile's `timeInApp.minutes` |
| `daysStudied` | distinct IST days with a **study** action (the `actions30d` tables). ⚠️ **not** `timeInApp.activeDays` (any request = "opened the app") |

The sitting rule now lives in ONE SQL builder (`sittingsCte`,
`src/modules/sme/services/sme-sittings.util.ts`) that both the snapshot and the list read,
so a row and the profile it links to cannot disagree. `actions30d` is kept. One extra
query per page (≤ 100 ids); a failure fails the request.

Staging proof (window 2026-10-04T12:00Z → 2026-11-03T12:00Z): the profile's `timeInApp` is
unchanged by the refactor (user 7a3f19d2: 8 sessions / 119 min / 6 days / median 8.6;
17ee1dee: 31 / 2051 / 13 / 17.6 — before = after); the list rows for the same users and
window: `sessions`/`minutesInApp` 8/119 and 31/2051 (= the profile); 17ee1dee
`daysStudied` 7 = hand SQL over the study tables (7 IST dates).

**Deviation from the brief:** the brief described the profile's days as coming "from
attempts/progress tables"; the profile's `activeDays` actually counts any request/event.
`daysStudied` follows the brief (study tables) and the difference is documented.

### B6.3 `/sme/analytics/*` dates are IST days

`startDate` / `endDate` on every route served by `SmeAnalyticsService` (dau, features,
users, signups, retention, premium-engagement, paywall-hits, heatmap, endpoints, content,
notifications) now go through `istDateRange` (`sme-date-range.util.ts`, the B1 rule):

- bare `YYYY-MM-DD` = an IST day: start inclusive from 00:00 IST, end = the **whole** IST
  day (< next 00:00 IST), so `startDate=endDate=D` is one day;
- an ISO instant is literal — the portal's `+05:30` workaround keeps working and lands on
  the same bounds;
- unparseable or inverted → **400** (it used to reach Postgres as an Invalid Date).

Also fixed: the rollup's DATE grain (`dau`, `endpoints`) compared the UTC date of the bound
(the day before an IST midnight) — it compares IST dates now; and `paywall-hits`, `heatmap`,
`premium-engagement` and the `/users` `actions30d` join ignored `endDate` — they honour it.
Without dates nothing moves (`until = now`). The insight routes (release-health,
notification-effectiveness, onboarding-funnel, question-quality) already used
`resolveIstWindow` and are unchanged.

Staging proof: signups for `2026-09-01`..`2026-11-03` as bare dates and as `+05:30` instants
render identical bounds (`2026-08-31 18:30` .. `2026-11-03 18:29:59.999` UTC) and both
return 6 = hand SQL over IST dates.

### B6.4 `platform=none` + segment count

- `GET /sme/users?platform=none` — users with **no active** push token on any platform.
  `platform` is validated (`ios|android|web|none`, else 400). It used to match the literal
  `'none'` and return 0.
- `GET /sme/notifications/segment/count?segment=&exam=` →
  `{ matchedUsers, segment, exam, examDefaulted }` — what `POST /sme/notifications/segment`
  would reach, **sending nothing**. Both routes call one method (`segmentAudience`:
  validation + `resolveSegment`). **`examDefaulted` is always `false`**: the send has no
  default exam (no `exam` = every user in every exam), so `true` would make the portal show
  "UPSC (default)" over an all-exams audience. Internal accounts are counted (the send
  reaches them).

Staging proof (read-only, 15 users): the real `countSegment` = an independent hand SQL for
every scope × cohort — no exam 5 / 0 / 10, `upsc-cse` 2 / 0 / 11, `appsc-group-1` 0 / 0 / 2
(premium / trial / free; staging has nobody in a trial today, so `trial` = 0 — B3.2 proved
`resolveSegment` at four instants); the real `listUsers` `platform=none` total 10 = hand SQL.

### B6.5 What the portal lanes must do (checklist)

1. **P2 engagement explorer** (`/sme/analytics/users`): drop "Actions"; show `daysStudied`,
   `sessions`, `minutesInApp` with the window label. Do not relabel `daysStudied` as "days
   active" — it is days with a study action.
2. **Every `/sme/analytics/*` call**: plain `YYYY-MM-DD` now works for `startDate`/`endDate`;
   the `+05:30` instants may stay. A bad date is now a 400 — show it, do not swallow it.
3. **P3 users list**: the Push-device filter can offer "No push device" → `platform=none`;
   remove the FastAPI guard that rejects it (`app/main.py` `_sme_users_params`: "there is
   no 'no push device' filter upstream").
4. **P4 push composer**: before Send, call `GET /sme/notifications/segment/count` with the same
   `segment` + `exam` and show "will reach N people (devices without push are counted but not
   reached)". Never label the un-scoped count "UPSC".

### B6.6 To verify after the staging deploy (HTTP)

- `GET /sme/analytics/users?days=30&search=<a staging user>` → the row's
  `sessions`/`minutesInApp` = `GET /sme/users/:id/snapshot?days=30` `timeInApp.sessions`/`minutes`
  (both trailing 30 days from now; the snapshot takes only `days`). Call them back to back —
  a sitting in progress between the two calls can move the numbers.
- `GET /sme/analytics/signups?startDate=2026-09-01&endDate=2026-11-03` and the same with
  `+05:30` instants → identical bodies; `?startDate=yesterday` → 400.
- `GET /sme/users?platform=none&includeInternal=1` → total 10 (staging today);
  `?platform=IOS` → 400.
- `GET /sme/notifications/segment/count?segment=premium` → `{ "matchedUsers": 5, "segment":
  "premium", "exam": null, "examDefaulted": false }`; `&exam=appsc-grp-1` → 400.
