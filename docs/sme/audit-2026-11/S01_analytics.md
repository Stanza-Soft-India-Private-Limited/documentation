# S01 — `/analytics` + the `/` cards built from `GET /sme/analytics` (Phase 1 verifier)

Read-only pass, 2026-11-03. Portal at `7b00daf` (v2.44.0). Backend `main` at `4ea16f2`. Between `89a072d` and HEAD only one commit touches this data path, `8b26a50`, and it is additive (`uiLanguage` on user rows). So the code read here is effectively what production runs (prod image `upsc:163`). No SQL was run. Every number below is a code-derived verdict, and each one comes with a reconcile query in §A.3.

Path keys: `M:` = `upsc-document-generator/app/main.py`; `A:` = `admin-ui/src/pages/Analytics.tsx`; `OPS:` / `AD:` / `CD:` = `admin-ui/src/components/features/dashboard/{AppOpsDashboard,AnalyticsDashboard,ContentDashboards}.tsx`; `UQ:` = `backend/src/modules/sme/services/sme-user.service.ts`; `TX:` = `…/sme-transaction.service.ts`; `PC:` = `backend/src/common/utils/premium-check.util.ts`; `AP:` = `backend/src/modules/payments/services/apple-payments.service.ts`; `PS:` = `backend/src/modules/payments/payments.service.ts`.

## A.0 The data path, verified

- UI `api.ts:2707–2710` `getSmeAnalytics(exam)` → `GET /sme/analytics?exam=` with a 60 s client timeout. React Query keys: `['sme-analytics', exam]` (A:198–202) and `['dash-sme-analytics']` (OPS:31, AD:12, CD:80). The dashboards **never pass `exam`**.
- M:5625–5875 makes four `_sme_fetch_all` calls:
  - users, orders and transactions all carry `params={"exam": exam}` (M:5631–5636);
  - webhooks carries `params={"source":"APPLE"}` and `max_pages=30` (M:5644–5646).
- **(July item 1) Confirmed fixed.** `source` travels in `params`, and `_sme_proxy` also merges any query string already on the path (M:4356–4366). The belt-and-braces filter `str(e.get("eventType") or "").startswith("apple.")` is present (M:5654). Our side matches: `/sme/webhooks` with `source=APPLE` resolves to `provider='apple' AND kind='webhook'` (TX:227–236), and the Apple handler writes `type = 'apple.<TYPE>[.<SUBTYPE>]'` (AP:835). Bug class 1 does not apply to this screen: it uses only `_sme_proxy`, never `_sme_proxy_raw`.
- **A new, different contamination exists. The Apple ledger includes sandbox notifications.**
  - Apple sends Sandbox and Production notifications to the same webhook.
  - The handler records each one in `payment_events` **before** its environment gate (AP:846–854). The gate then marks foreign-environment events `result='ignored'` (AP:870–879).
  - `/sme/webhooks` never filters on `result` (TX:227–256), and the portal never reads it either (M:5654, 5791–5814).
  - Result: `isTest` is excluded from orders and payments (TX:69–71, 93, 162), but **not** from events.
- **`_sme_fetch_all` (bug classes 2 and 8)**, re-confirmed at M:5524–5579:
  - 100 rows × 400 pages = a 40,000-row ceiling per corpus.
  - A failure on page 1 raises → `/sme/analytics` returns the upstream error → A:215–224 shows "Could not load analytics".
  - A failure on a later page is counted and its rows are dropped (M:5559–5564).
  - `complete = not truncated and failedPages == 0` (M:5577). It never compares `fetched` with `total`.
  - The scan is OFFSET paging over `createdAt DESC` (UQ:126–130, TX:174–178) while the table is live. A signup that lands during the ~65-page user scan shifts every later page by one: one row is counted twice, one is never fetched, and `complete` still reads true.
  - `coverage.complete` covers users, orders and payments only. The Apple leg (3,000-row cap) sits in `coverage.appleEvents`, which **no UI renders**.
- **What happens past 40,000 users or on a failed page:**
  - `/analytics` shows the amber banner (A:281–302) with "fetched X of Y (hit the page ceiling) · N page(s) failed". Today the user base is ~7k, so this is not biting yet.
  - **None of the three `/` dashboards renders `coverage` at all.** OPS, AD and CD would show a partial count as "Total registered".
  - `AnalyticsDashboard` and `AdminDashboard` don't even show an error when the call fails: their KPIs fall back to "—" (AD:12–31, CD:80–107; `nfmt(undefined)='—'`, shared.tsx:9–14). Only OPS:74 shows `DataError`.
- **Window: every figure is ALL-TIME.** The only exceptions are the two month series, which keep the last 12 months that have rows, bucketed by UTC month. Nothing on the page or the cards says "all time" or "all exams" (dashboards).
- **Internal accounts: no exclusion anywhere on this path** (UQ:64–158, TX:81–194, M:5625–5875). Bug class 4 applies to every user and money number. The App Store review account is `@stanzasoft.com`, so the domain rule covers it. Team members on personal Gmail or Apple relay addresses cannot be caught by any rule.

### A.1 TRUST — per-number table

Plane is **C** (billing/user tables) for every row. In "Their transformation", a bare `M:` line is the Python aggregate. Prod-SQL ids refer to §A.3.

| # | Number as labelled (UI file:line) | Our definition (route; file:line; SQL shape) | Their transformation (M:line; UI line) | Window + TZ | Exclusions applied | Draft verdict + why | SQL | Sev |
|---|---|---|---|---|---|---|---|---|
| 1 | **Total Users** (A:252–256); "App users · Total registered" (OPS:78, AD:28, CD:105) | `/sme/users` row = one `user_auth` row, all statuses (UQ:125–143) | `len(users)` M:5657 | all-time; none | none (internal, demo and sandbox testers all included) | **TRUE as a floor of `user_auth`**, not "real users". Can be off by ±n under concurrent signups (offset drift); `complete` can't see that. With `exam` = retro-attributed by `active_exam_id` (NULL ≡ upsc-cse, `sme-exam-scope.util.ts:26–37`) | Q1 | 🟠 (internal not excluded; `/` has no floor banner) |
| 2 | "x% onboarded" (A:254); **Onboarded** % + "N completed" (OPS:84, app_admin_users only); "Onboarding" (AD:48) | `onboardingCompleted: u.userProfile?.onboardingCompleted ?? null` (UQ:755) | `sum(onboardingCompleted)` M:5659 ÷ total M:5838 | all-time current state | none | **TRUE.** The key exists on the row (UQ:755), so the count is not silently zero. Users with no profile count as not onboarded. | Q1 | 🟡 |
| 3 | **Premium Access** + "x% of users" (A:257–262) | `isPremium = isPremiumUser(mirror, trialDays of active exam)` (UQ:720–728, PC:117–129) = (SUBSCRIBED and not expired) OR (ACTIVE and inside the trial) | `sum(isPremium)` M:5658; `conversionRate` M:5836 | all-time; "now" = request time | none | **TRUE** with its stated definition (A:260). **(July item 5) confirmed:** row-level `isPremium` includes trials. Includes MANUAL comps, sandbox testers (sandbox verify **does grant**: AP:367, 565–590; only Meta CAPI is skipped, AP:531–534) and internal accounts. **With `exam` it is MISLABELLED:** people whose *active* exam is X who are premium in *any* exam (person mirror, `docs/sme/SME_PORTAL_API.md` §1.1 says so). The per-exam answer (`entitlements[]`, UQ:734–738, 775) is fetched but ignored. | Q2, Q2b | 🟠 |
| 4 | **"Premium" + "x% conversion"** (OPS:79, AD:30, CD:106) | same as #3 | `users.premium`, `users.conversionRate` | all-time | none | **MISLABELLED.** This is premium *access* including in-window trials, labelled "conversion" with no definition tooltip, on the screen shareholders land on. `/analytics` labels the same number "Premium Access … % of users" (A:258–260). | Q2 | 🟠 |
| 5 | **Paying Subscribers** + "x% paid · N on trial" (A:263–268) | `premiumState === 'Premium'` = SUBSCRIBED and (no expiry or expiry > now) (PC:302–318) | `_tally(users,'premiumState')['Premium']` M:5839; % computed in UI A:247–249 | all-time | none | **MISLABELLED.** The doc calls it "the honest revenue-side number" (`docs/sme/SME_METRICS_DECODER.md` §1), but it counts anyone SUBSCRIBED: MANUAL comps (PS:1592–1612 sets PAID + premium), **sandbox testers** (AP:565–590) and internal accounts. Q2b separates out the people with no real-money PAID order. | Q2b | 🟠 |
| 6 | **On trial** (OPS:80) | `premiumState === 'Trial'` (PC:319–321) | `byPremiumState['Trial']` | now | none | **TRUE** (trial length is per active exam, UQ:720–723). Includes internal test signups. | Q2 | 🟡 |
| 7 | **Subscription funnel** bars (A:324–327) | 5-way `derivePremiumState` (PC:302–325) | `_tally(users,'premiumState')` M:5839 | all-time | none | **TRUE** to the definition. Exam-filtered = retro-attributed people with *person-level* state (see #3). | Q2 | 🟡 |
| 8 | **User status** (A:328–331) | raw `user_auth.status` (UQ:748) | `_tally(users,'status')` M:5840 | stored column | none | **TRUE** (the stored column, which can be stale; documented). | Q3 | ⚪ |
| 9 | **Sign-in provider** (A:332–335) | `user_auth.provider` (UQ:746); null → "Unknown" (M:5587–5588) | `_tally` M:5841 | all-time | none | **TRUE.** | Q3 | ⚪ |
| 10 | **"N premium subscription(s) expiring within 7 days"** (A:313–318) | `isPremium` + `premiumExpiresAt` (UQ:749, 753) | M:5669–5685: `isPremium` and `today ≤ expiry[:10] ≤ today+7`, where `today = date.today()` in **UTC** on App Runner | next 7 days, UTC date | none | **MISLABELLED.** Most of these are auto-renewing Apple or Razorpay subscriptions whose expiry is just the next renewal date: "expiring" reads as churn. Trials never appear (no `premiumExpiresAt`). Includes MANUAL, sandbox and internal. The UTC date can be off by one versus IST. | Q5 | 🟡 |
| 11 | **Revenue (captured)** (A:269–273; OPS:82, AD:31, CD:107) | payments rows, `amount = paise/100` (TX:307); test orders excluded (TX:93) | CAPTURED payments (M:5688) → drop non-INR storefront (M:5704–5711) → by source (M:5719–5723) → sum excluding MANUAL (M:5727–5732) | all-time | **isTest ✓, MANUAL ✓, non-INR ✓ (July item 4 confirmed)**; internal ✗; **refunds ✗** | **MISLABELLED: it is gross of refunds.** No code ever sets `payments.status = 'REFUNDED'` (`rg REFUNDED` finds only the DTO enum, `sme-transaction.dto.ts:19`). Apple REFUND/REVOKE writes a `refunds` row (AP:1104–1125). Razorpay refunds go to `refunds` too. So a refunded payment stays CAPTURED and stays in this total. Internal-account purchases are included too. | Q6, Q7 | 🟠 |
| 12 | "N payments" (A:271) | as #11 | `len(captured)` M:5851 | all-time | **MANUAL and non-INR NOT excluded** | **MISLABELLED / inconsistent.** The count includes MANUAL and non-INR payments while the ₹ beside it excludes them. | Q6 | 🟡 |
| 13 | **"x% success"** (A:271; OPS:82, AD:31, CD:107) | payments only get a row when they succeed: the verify path writes CAPTURED/AUTHORIZED/PENDING (PS:385–410), the webhook upserts CAPTURED/AUTHORIZED (PS:1239–1275), Apple and subscription charges write CAPTURED (AP:491–503, 1266–1278; `razorpay-subscriptions.service.ts:761–769`); the failure handler only **updates** an existing row (PS:968–985) | `captured / len(payments)` M:5853 | all-time | isTest ✓ | **MISLABELLED (≈100% by construction).** A failed checkout almost never has a payment row, so this cannot show checkout failure. The decoder's "denominator includes CREATED/FAILED attempts" is false (§D). The real failure signal is order CREATED/FAILED (#17) or the purchase funnel. | Q6, Q10 | 🟠 |
| 14 | **Payment method** bars + "N captured · N failed · N total" (A:344–352) | `payments.method` (TX:310); MANUAL = `'manual'` (PS:1612), Apple = `'apple_iap'` (AP:497) | `_tally(captured,'method')` M:5690 | all-time | isTest ✓ | **TRUE** for the bars (manual shows as its own bar). "failed" undercounts (see #13). | Q8 | 🟡 |
| 15 | **Revenue by source** + caption "Captured ₹ split across Apple · Razorpay · Manual" (A:353–360) | as #11 | `captured_revenue_by_source` M:5719–5723; source = the order's `paymentSource`, else `apple_iap` → APPLE, else Unknown (M:5713–5717) | all-time | isTest ✓, non-INR ✓, MANUAL **shown** | **MISLABELLED (minor).** The MANUAL bar is the *list price stamped on comps* (PS:1571, 1595), not captured money, yet the caption calls it "Captured ₹". APPLE and RAZORPAY: TRUE. | Q6 | 🟡 |
| 16 | **Revenue by exam** (A:119–169, 361) | order `examId` (TX:121, 364) | M:5736–5745, only when no exam is selected; MANUAL excluded; NULL → `unresolved` | all-time | isTest ✓, MANUAL ✓, non-INR ✓ | **TRUE**, but gross of refunds (#11). With an exam selected, the card correctly refuses (A:149–152). | Q9 | 🟡 |
| 17 | **Order status** bars (A:336–339) | `orders.status` CREATED/PAID/FAILED/CANCELLED; `isTest` excluded (TX:162) | `_tally(orders,'status')` M:5866 | all-time | isTest ✓; MANUAL ✗ (PAID); internal ✗ | **TRUE.** CREATED = abandoned checkouts (documented). PAID includes MANUAL comps. | Q10 | ⚪ |
| 18 | **Plan mix** (A:340–343) | `orders.planType` | `_tally(orders,'planType')` M:5867 | all-time | isTest ✓ only | **MISLABELLED.** Counts every order of every status (abandoned CREATED, FAILED, MANUAL), not sales. NULL plan (unresolved Apple) → "Unknown". | Q10 | 🟡 |
| 19 | **"N paid order(s) need reconcile (premium not granted)"** (A:307–312) | PAID with `premiumGrantedAt` NULL. Our row already carries `reconcileHint` GRANT_PENDING vs UNRESOLVED_EXAM (TX:344–351, 382) | `needs_reconcile` M:5756 over `paid_orders_all` (non-INR included) | all-time | isTest ✓ | **TRUE (July item 3).** Every PAID path stamps `premiumGrantedAt`: MANUAL PS:1602, Apple AP:402/1257 when the product resolved, Razorpay via the grant at PS:1454. A non-zero value is a real alert. But it merges our two hints that need *opposite* fixes, and with an exam selected UNRESOLVED_EXAM (exam_id NULL) **vanishes from every exam view** (TX:44–51). Q10c checks the doc's "pre-2026-06-16 rows count forever" claim. | Q10c | 🟡 |
| 20 | **Revenue by month (paid orders)** bars (A:435–444) | `orders.amount/100`, `createdAt` (TX:365, 383) | `paid_orders` = PAID minus non-INR (M:5754); bucket = `createdAt[:7]` **UTC** (M:5757–5762); last 12 months that have rows | 12 months, UTC month of **order creation** | isTest ✓, non-INR ✓, **MANUAL ✗** | **WRONG.** Includes MANUAL comps at list price, so the bars don't add up to the "Revenue (captured)" headline, which excludes them. Also order-created (not captured), UTC month (IST 00:00–05:30 on the 1st lands in the previous month), gross of refunds, and months with no rows are skipped rather than drawn at zero. | Q11 | 🔴 |
| 21 | **New users by month** (A:426–434) | `user_auth.createdAt` | `createdAt[:7]` UTC (M:5661–5666), last 12 months that have rows | 12 months, **UTC** | none | **TRUE except the TZ** (bug class 5: IST signups 00:00–05:29 on the 1st land in the prior month). Internal included. Exam = retro-attributed. | Q4 | 🟡 |
| 22 | **Signups (this month)** "Latest month" (AD:15, 29) | as #21 | the last element of `signupsByMonth` | the latest UTC month **with any signup** | none | **MISLABELLED (edge).** Shows the previous month whenever the current UTC month has no rows yet (e.g. 1st, 00:00–05:30 IST). Otherwise TRUE. | Q4 | ⚪ |
| 23 | **Apple revenue** (A:374–382) | PAID APPLE orders, `amount` (INR) | M:5785–5790, non-INR excluded and shown separately (A:377–381) | all-time | isTest ✓, non-INR ✓ | **TRUE**, but gross of refunds, and on an *order* basis while "Revenue by source → APPLE" (#15) is on a *payment* basis. The two must match; Q13 checks. | Q13 | 🟡 |
| 24 | **Subscribers** (Apple) (A:383–386) | distinct `userId` over PAID APPLE orders | M:5797 | **lifetime** | isTest ✓ | **MISLABELLED.** Lifetime distinct buyers, read as current subscribers. Current state lives in `apple_subscriptions` / `user_exam_entitlements`, neither of which is in this payload. | Q13 | 🟠 |
| 25 | **Paid orders (incl. renewals)** (A:387–390) | PAID APPLE orders (one per billing period, AP:1235–1278) | M:5786, 5800 | all-time | isTest ✓ | **TRUE.** | Q13 | ⚪ |
| 26 | **Apple Plan mix (MONTHLY / ANNUAL)** (A:405–408) | `orders.planType` | `_tally(apple_orders,'planType')` M:5804, **all statuses** | all-time | isTest ✓ | **MISLABELLED (minor).** Includes non-PAID Apple orders; NULL plan → "Unknown". | Q13 | ⚪ |
| 27 | **Apple events recorded** (A:391–401) | `payment_events WHERE provider='apple' AND kind='webhook'` (TX:227–236), newest first | 30 pages × 100 (M:5644–5646), `apple.*` only (M:5654), `len` (M:5807) | the newest ≤3,000 events, received_at | **sandbox (`result='ignored'`) NOT excluded**; truncation not shown | **WRONG.** (a) Counts sandbox notifications, which the backend itself ignores (AP:870–879). Sandbox subscriptions auto-renew every few minutes, so the volume can dominate. (b) Silently capped at 3,000; `coverage.appleEvents` is never rendered. (c) With `exam` selected it is still **all exams** (the webhooks call carries no exam; TX:258–262 and `SME_APPLE_IAP_API.md` §3 say exam is ignored by design). The Razorpay leak (July item 1) is fixed. | Q14 | 🔴 |
| 28 | **New / Renewals / Billing issues / Expired / Refunds** (A:409–418) | `type` prefixes (AP:835) | `_evt_count` prefixes M:5793–5813 | as #27 | sandbox ✗ | **WRONG**, same as #27(a–c). **Renewals** is the most exposed (sandbox DID_RENEW storms). Pre-gate sandbox rows were stored as `processed`, so Q14 decodes the JWS `data.environment` to separate them. | Q14 | 🔴 |
| 29 | Non-INR notes (A:104–113, 358, 377–381, 442) | `storefrontCurrency`, `chargedAmount` (TX:377–378) | `_add_foreign_charge` M:5617–5622 | all-time | isTest ✓ | **TRUE.** | Q6 | ⚪ |
| 30 | Partial-data banner (A:281–302) | `total` from our list response (UQ:142, TX:128/184) | `coverage` M:5820–5829 | n/a | n/a | **MISLABELLED as a guarantee.** Silent when offset drift changes rows without changing `fetched`, and blind to the Apple leg (bug class 8). Absent on `/` entirely. | — | 🟠 |
| 31 | **"Trials expiring soon"** list (OPS:41–52, 91–115). Different payload, rendered beside these cards | `/sme/users` rows sorted `createdAt DESC` (UQ:130) | portal `premiumState=Trial` → full scan + local filter, keeps the **first 50 = the 50 newest trial users** (M:4560–4573); UI then sorts those 50 by `trialDaysLeft` and shows 6 (OPS:49–52) | now | none (internal shown) | **WRONG.** With hundreds of trialists, the 50 newest signed up within the last day or so (~13 days left). The people actually expiring are never on the list a sales user works from. | Q16 | 🔴 |

### A.2 What the `exam` selector does to each card (`/analytics` only; `/` is always all-exams)

| Card | Axis used under `?exam=X` | Effect |
|---|---|---|
| Total Users, onboarded, funnel, status, provider, new users by month | PERSON: `active_exam_id`, NULL ≡ upsc-cse (UQ:86–91) | Retro-attributed: a June signup who switched to APPSC last week is a June APPSC signup. |
| Premium Access, Paying Subscribers, Expiring ≤7d | PERSON *population* × **person-level** premium mirror | **Cross-exam premium shown as if per-exam.** A UPSC buyer whose active exam is APPSC counts as "APPSC premium". Not labelled. |
| Revenue (captured), success, method, by source, order status, plan mix, reconcile, revenue by month, Apple orders, revenue, subscribers | BOUGHT: `orders.exam_id = X` (TX:44–51, 92, 161) | Correct per exam. NULL-exam (UNRESOLVED) money disappears from **every** exam view, `upsc-cse` included (the opposite NULL rule to users). The UI never says so. |
| Revenue by exam | n/a | Hidden with an explanation (A:149–152). Correct. |
| Apple events + 5 counters | **none**: the webhooks call carries no exam (M:5644–5646) | **Cross-exam totals shown inside an exam-filtered page.** 🟠 |

So a page filtered to APPSC mixes three populations on one screen: people who chose APPSC, people premium in any exam, and money spent on APPSC.

### A.3 Reconcile SQL (read-only, production Postgres; NOT run)

Prepend **Q0** to every query. Columns follow `prisma/schema.prisma`:
- `user_auth."createdAt"`, `premium_expires_at`, `trial_ends_at` (timestamptz);
- `user_profiles."userId"`, `"onboardingCompleted"`, `active_exam_id`;
- `orders."userId"`, `payment_source`, `plan_type`, `exam_id`, `is_test`, `storefront_currency`, `premium_granted_at`, `created_at`, `amount` (paise);
- `payments.order_id`, `amount`, `status`, `method`;
- `payment_events.provider`, `kind`, `type`, `result`, `raw_body`;
- `refunds.payment_id`, `provider_payment_id`;
- `apple_subscriptions.*`.

Prisma `DateTime` without `@db.Timestamptz` is `timestamp without time zone`, stored as UTC.

How each query reports its result:
- **portal-equivalent** = what the screen shows: internal accounts kept, `is_test` orders dropped.
- **real** = internal accounts and sandbox testers also excluded.

For an exam-scoped run:
- users: add `AND (active_exam_id IS NULL OR active_exam_id = 'upsc-cse')` for upsc-cse, or `AND active_exam_id = '<slug>'` for any other exam;
- orders and payments: add `AND o.exam_id = '<slug>'`.

```sql
-- Q0 shared CTEs
WITH internal_users AS (
  SELECT id FROM user_auth
  WHERE lower(email) LIKE '%@stanzasoft.com' OR lower(email) LIKE '%@stanzasoft.ai'
     OR lower(email) ~ '@prepmonkey\.'      OR lower(email) LIKE '%@designasylum.in'
),
sandbox_users AS (SELECT DISTINCT "userId" AS id FROM orders WHERE is_test),
excluded AS (SELECT id FROM internal_users UNION SELECT id FROM sandbox_users),
us AS (
  SELECT u.id, u.status, u.provider, u."createdAt",
         u.premium_expires_at                      AS pe_utc_wall,
         (u.premium_expires_at AT TIME ZONE 'UTC') AS pe,
         COALESCE(u.trial_ends_at,
                  (u."createdAt" AT TIME ZONE 'UTC') + make_interval(days => COALESCE(td.days, 14))) AS trial_end,
         COALESCE(p."onboardingCompleted", false) AS onboarded,
         p.active_exam_id,
         (u.id IN (SELECT id FROM excluded)) AS is_internal
  FROM user_auth u
  LEFT JOIN user_profiles p ON p."userId" = u.id
  LEFT JOIN LATERAL (            -- per-exam trial_policy, as ExamsService.getPersonTrialDays
    SELECT (e->>'days')::int AS days
    FROM exams x, jsonb_array_elements(x.trial_policy::jsonb) e
    WHERE x.id = COALESCE(p.active_exam_id, 'upsc-cse')
      AND (e->>'days')::int >= 0
      AND (e->>'effectiveFrom')::timestamptz <= (u."createdAt" AT TIME ZONE 'UTC')
    ORDER BY (e->>'effectiveFrom')::timestamptz DESC LIMIT 1
  ) td ON true
),
uss AS (
  SELECT *, CASE
    WHEN status = 'SUBSCRIBED' AND (pe IS NULL OR pe > now()) THEN 'Premium'
    WHEN status = 'ACTIVE' AND now() < trial_end              THEN 'Trial'
    WHEN pe IS NOT NULL                                       THEN 'Churned'
    WHEN now() >= trial_end                                   THEN 'Trial Ended'
    ELSE 'Downloaded' END AS premium_state
  FROM us
)
```

```sql
-- Q1  Total users / onboarded  (#1, #2)
SELECT count(*) AS total, count(*) FILTER (WHERE NOT is_internal) AS total_real,
       count(*) FILTER (WHERE onboarded) AS onboarded,
       count(*) FILTER (WHERE onboarded AND NOT is_internal) AS onboarded_real
FROM uss;

-- Q2  Premium Access (= Premium + Trial), Paying, Trial, funnel  (#3–#7)
SELECT premium_state, count(*) AS portal_equiv, count(*) FILTER (WHERE NOT is_internal) AS real
FROM uss GROUP BY ROLLUP (premium_state) ORDER BY 1;

-- Q2b Paying Subscribers that are NOT backed by real money  (#5)
SELECT count(*) AS paying_portal,
       count(*) FILTER (WHERE is_internal) AS paying_internal_or_sandbox,
       count(*) FILTER (WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o."userId" = uss.id
                          AND o.status = 'PAID' AND NOT o.is_test AND o.payment_source <> 'MANUAL')) AS paying_without_real_paid_order,
       count(*) FILTER (WHERE NOT is_internal AND EXISTS (SELECT 1 FROM orders o WHERE o."userId" = uss.id
                          AND o.status = 'PAID' AND NOT o.is_test AND o.payment_source <> 'MANUAL')) AS paying_real
FROM uss WHERE premium_state = 'Premium';

-- Q3  User status / provider  (#8, #9)
SELECT 'status' AS dim, status::text AS k, count(*), count(*) FILTER (WHERE NOT is_internal) FROM uss GROUP BY 2
UNION ALL
SELECT 'provider', COALESCE(provider, 'Unknown'), count(*), count(*) FILTER (WHERE NOT is_internal) FROM uss GROUP BY 2
ORDER BY 1, 3 DESC;

-- Q4  Signups by month, UTC (portal) vs IST (correct), last 13 months  (#21, #22)
SELECT 'utc' AS tz, to_char("createdAt", 'YYYY-MM') AS m, count(*) AS n, count(*) FILTER (WHERE NOT is_internal) AS n_real
FROM uss WHERE "createdAt" >= now() - interval '13 months' GROUP BY 2
UNION ALL
SELECT 'ist', to_char(("createdAt" AT TIME ZONE 'UTC') AT TIME ZONE 'Asia/Kolkata', 'YYYY-MM'), count(*), count(*) FILTER (WHERE NOT is_internal)
FROM uss WHERE "createdAt" >= now() - interval '13 months' GROUP BY 2
ORDER BY 1, 2;

-- Q5  "Expiring within 7 days"  (#10)   portal uses the UTC calendar date of both sides
SELECT count(*) AS portal_equiv,
       count(*) FILTER (WHERE NOT is_internal) AS real,
       count(*) FILTER (WHERE EXISTS (SELECT 1 FROM user_exam_entitlements e
                         WHERE e.user_id = uss.id AND e.auto_renew IS TRUE AND e.revoked_at IS NULL)) AS of_which_auto_renewing,
       count(*) FILTER (WHERE EXISTS (SELECT 1 FROM apple_subscriptions a
                         WHERE a.user_id = uss.id AND a.auto_renew IS TRUE)) AS of_which_apple_auto_renew
FROM uss
WHERE premium_state IN ('Premium', 'Trial')
  AND pe_utc_wall::date BETWEEN (now() AT TIME ZONE 'UTC')::date AND (now() AT TIME ZONE 'UTC')::date + 7;

-- Q6  Revenue (captured), counts, success, by source, non-INR  (#11–#13, #15, #29)
SELECT COALESCE(o.payment_source::text, 'ALL') AS source,
  count(*)                                          AS payments_total,
  count(*) FILTER (WHERE p.status = 'CAPTURED')     AS captured_n,
  count(*) FILTER (WHERE p.status = 'FAILED')       AS failed_n,
  count(*) FILTER (WHERE p.status IN ('PENDING','AUTHORIZED')) AS pending_or_auth_n,
  round(sum(p.amount) FILTER (WHERE p.status = 'CAPTURED'
        AND (o.storefront_currency IS NULL OR upper(o.storefront_currency) = 'INR')) / 100.0, 2) AS captured_inr,
  round(sum(p.amount) FILTER (WHERE p.status = 'CAPTURED'
        AND (o.storefront_currency IS NULL OR upper(o.storefront_currency) = 'INR')
        AND o."userId" NOT IN (SELECT id FROM internal_users)) / 100.0, 2) AS captured_inr_real,
  count(*) FILTER (WHERE p.status = 'CAPTURED' AND upper(COALESCE(o.storefront_currency, 'INR')) <> 'INR') AS non_inr_n
FROM payments p JOIN orders o ON o.id = p.order_id
WHERE NOT o.is_test
GROUP BY ROLLUP (o.payment_source) ORDER BY 1;
-- headline "Revenue (captured)" = sum(captured_inr) over source <> 'MANUAL'

-- Q7  Refunds sitting inside "captured"  (#11, #16, #23)
SELECT r.provider, r.status, count(*) AS n, sum(r.amount) AS amount_as_stored FROM refunds r GROUP BY 1, 2;
SELECT o.payment_source, count(*) AS refunded_but_captured, round(sum(p.amount) / 100.0, 2) AS inr
FROM payments p JOIN orders o ON o.id = p.order_id
WHERE p.status = 'CAPTURED' AND NOT o.is_test AND o.payment_source <> 'MANUAL'
  AND EXISTS (SELECT 1 FROM refunds r WHERE r.payment_id = p.id
              OR r.provider_payment_id IN (p.razorpay_payment_id, p.apple_transaction_id))
GROUP BY 1;

-- Q8  Payment method  (#14)
SELECT COALESCE(p.method, 'Unknown') AS method, count(*)
FROM payments p JOIN orders o ON o.id = p.order_id
WHERE NOT o.is_test AND p.status = 'CAPTURED' GROUP BY 1 ORDER BY 2 DESC;

-- Q9  Revenue by exam  (#16)
SELECT COALESCE(o.exam_id, 'unresolved') AS exam, round(sum(p.amount) / 100.0, 2) AS inr,
       round(sum(p.amount) FILTER (WHERE o."userId" NOT IN (SELECT id FROM internal_users)) / 100.0, 2) AS inr_real
FROM payments p JOIN orders o ON o.id = p.order_id
WHERE NOT o.is_test AND p.status = 'CAPTURED' AND o.payment_source <> 'MANUAL'
  AND (o.storefront_currency IS NULL OR upper(o.storefront_currency) = 'INR')
GROUP BY 1 ORDER BY 2 DESC;

-- Q10 Orders by status / plan / source  (#17, #18)
SELECT status, payment_source, COALESCE(plan_type, 'Unknown') AS plan, count(*)
FROM orders WHERE NOT is_test GROUP BY ROLLUP (status, payment_source, plan_type) ORDER BY 1, 2, 3;

-- Q10c Reconcile, split the way our reconcileHint splits it  (#19)
SELECT count(*) FILTER (WHERE premium_granted_at IS NULL)                         AS needs_reconcile_portal,
       count(*) FILTER (WHERE premium_granted_at IS NULL AND exam_id IS NOT NULL) AS grant_pending,
       count(*) FILTER (WHERE premium_granted_at IS NULL AND exam_id IS NULL)     AS unresolved_exam,
       count(*) FILTER (WHERE premium_granted_at IS NULL AND created_at < '2026-06-16') AS predates_field
FROM orders WHERE status = 'PAID' AND NOT is_test;

-- Q11 Revenue by month (paid orders): portal (incl. MANUAL, UTC) vs excl. MANUAL, IST  (#20)
SELECT to_char(created_at, 'YYYY-MM') AS utc_month,
       round(sum(amount) / 100.0, 2)                                           AS portal_equiv,
       round(sum(amount) FILTER (WHERE payment_source = 'MANUAL') / 100.0, 2)  AS of_which_manual
FROM orders
WHERE status = 'PAID' AND NOT is_test AND (storefront_currency IS NULL OR upper(storefront_currency) = 'INR')
  AND created_at >= now() - interval '13 months'
GROUP BY 1 ORDER BY 1;
SELECT to_char((created_at AT TIME ZONE 'UTC') AT TIME ZONE 'Asia/Kolkata', 'YYYY-MM') AS ist_month,
       round(sum(amount) / 100.0, 2) AS paid_excl_manual
FROM orders
WHERE status = 'PAID' AND NOT is_test AND payment_source <> 'MANUAL'
  AND (storefront_currency IS NULL OR upper(storefront_currency) = 'INR')
  AND "userId" NOT IN (SELECT id FROM internal_users)
  AND created_at >= now() - interval '13 months'
GROUP BY 1 ORDER BY 1;

-- Q12 Integrity: PAID orders with no CAPTURED payment (the decoder's "cheapest check")
SELECT o.payment_source, count(*) AS paid_without_capture, round(sum(o.amount) / 100.0, 2) AS inr
FROM orders o
WHERE o.status = 'PAID' AND NOT o.is_test
  AND NOT EXISTS (SELECT 1 FROM payments p WHERE p.order_id = o.id AND p.status = 'CAPTURED')
GROUP BY 1;

-- Q13 Apple block  (#23–#26): orders basis vs payments basis must agree
SELECT count(*)                                       AS apple_orders,
       count(*) FILTER (WHERE status = 'PAID')        AS apple_paid,
       round(sum(amount) FILTER (WHERE status = 'PAID'
             AND (storefront_currency IS NULL OR upper(storefront_currency) = 'INR')) / 100.0, 2) AS apple_revenue_orders,
       count(DISTINCT "userId") FILTER (WHERE status = 'PAID') AS subscribers_lifetime,
       (SELECT count(DISTINCT a.user_id) FROM apple_subscriptions a
          WHERE a.expires_date > (now() AT TIME ZONE 'UTC') AND lower(COALESCE(a.environment, 'production')) = 'production') AS subscribers_current
FROM orders WHERE payment_source = 'APPLE' AND NOT is_test;
SELECT plan_type, status, count(*) FROM orders WHERE payment_source = 'APPLE' AND NOT is_test GROUP BY 1, 2;

-- Q14 Apple events: sandbox share, by type, and the 3,000-row cap  (#27, #28)
WITH ev AS (
  SELECT type, result, received_at,
    convert_from(decode(rpad(translate(split_part(raw_body, '.', 2), '-_', '+/'),
                 ((length(split_part(raw_body, '.', 2)) + 3) / 4) * 4, '='), 'base64'), 'UTF8')::jsonb
      #>> '{data,environment}' AS env
  FROM payment_events WHERE provider = 'apple' AND kind = 'webhook' AND type LIKE 'apple.%'
)
SELECT COALESCE(env, '?') AS env, result, split_part(type, '.', 2) AS notification_type, count(*)
FROM ev GROUP BY ROLLUP (1, 2, 3) ORDER BY 1, 2, 4 DESC;
-- cap check: SELECT count(*) > 3000 FROM payment_events WHERE provider='apple' AND kind='webhook';

-- Q16 Trials actually expiring soonest vs what the dashboard shows  (#31)  (PII: count/compare, don't paste emails)
SELECT id, trial_end, ceil(extract(epoch FROM trial_end - now()) / 86400) AS days_left
FROM uss WHERE premium_state = 'Trial' AND NOT is_internal ORDER BY trial_end ASC LIMIT 6;
SELECT id, trial_end FROM (
  SELECT id, trial_end, "createdAt" FROM uss WHERE premium_state = 'Trial' ORDER BY "createdAt" DESC LIMIT 50
) newest50 ORDER BY trial_end ASC LIMIT 6;   -- portal-equivalent
```

## B. MEANING (six shareholder questions)

| Q | Answered here? | What is missing | Data already persists? |
|---|---|---|---|
| (1) Feature use + repeat rate | **No.** This screen has no usage. | everything | plane A tables (Engagement's job) |
| (2) Per-exam and per-language funnels | **Partially.** The exam selector gives a premium-state funnel, but on mixed axes (§A.2), with person-level premium and retro-attribution. **No language at all.** | per-exam *entitlement* premium (the row's `entitlements[]` is fetched and ignored); language split | **Yes**: `user_exam_entitlements` (exam); `user_profiles.ui_language` (already on every `/sme/users` row as `uiLanguage`, UQ:771) and `user_profiles.medium` (not on the list row). |
| (3) Where buyers stop in checkout | **Partially, and misleadingly.** Order CREATED/FAILED counts exist; "x% success" looks like a checkout success rate and isn't (#13). | step-by-step drop-off | `orders` (CREATED/FAILED), `payment_events`; the purchase funnel lives on `/insights/purchase-funnel` (a different screen) |
| (4) Question quality | No | — | n/a here |
| (5) Notification outcomes | No | — | n/a here |
| (6) Support view of one user incl. Razorpay mandates | No (aggregate screen) | — | n/a here |

"So what" gaps on this screen:
- no period-over-period comparison: everything is all-time;
- refunds are not netted;
- no current vs lifetime split for Apple subscribers;
- "Paid orders vs payments" (Q12), the integrity check the decoder recommends, cannot be read off the page: `orders.paid` is computed but never rendered.

## C. GAP

| Item | Source | State |
|---|---|---|
| `reconcileHint` (GRANT_PENDING vs UNRESOLVED_EXAM) on order rows | TX:382; `SME_APPLE_IAP_API.md` §2.2 | proxied-only: fetched, then recomputed as one bucket (M:5756) |
| `entitlements[]` per user row (per-exam live premium) | UQ:775; `SME_PORTAL_API.md` §1.1 | proxied-only: fetched, unused, so the exam view has no per-exam premium |
| `uiLanguage`, `activeExamId`, `trialExamId` on user rows | UQ:731, 767, 771 | proxied-only: never tallied |
| `offerCode`, `listAmount` on orders (campaign vs list revenue, discount given) | TX:369–370 | proxied-only |
| `result` on webhook rows (`processed`/`orphan`/`error`/**`ignored`**) | TX:285 | proxied-only: not used to drop sandbox `ignored` rows |
| `coverage.appleEvents` | M:5824 | proxied-only: never rendered, excluded from `complete` |
| `orders.paid`, `orders.total`, `orders.bySource`, `paidOrdersValue`, `paidValueBySource`, `paidValueByExam`, `revenue.manualRevenue`, `apple.events.byType` | M:5845–5873 | proxied-only: computed, never rendered |
| `coverage` banner on `/` dashboards | A:281 only | absent on OPS/AD/CD |
| Error state on AD and CD when `/sme/analytics` fails | OPS:74 only | absent (silent "—") |
| Refund totals | no SME aggregate route (only the per-user timeline, `sme-activity.service.ts:872`) | absent on our side: netting needs a backend change (flag only) |
| `/sme/webhooks` `startDate`/`endDate` | `SME_APPLE_IAP_API.md` §3 | not used (cap by pages instead) |
| Stale copy: "Read-only · derived from Razorpay + Apple data" (A:448) | — | page also includes MANUAL rows (bars #14, #15, #17, #20) |
| Stale copy: dashboards' "Total registered" / "conversion" | OPS:78–79, AD:28–30, CD:105–106 | no "all exams" or "incl. trials" qualifier |
| Portal calls to changed or deprecated routes | — | none: all four routes are current |

## D. Contradictions (code vs docs or brief)

1. **`SME_METRICS_DECODER.md` §1 and §8 (items 1–2 under "Still open") are stale.** They say the portal caps at 6,000 rows (now 40,000, M:5520) and that "Events recorded" includes Razorpay rows (fixed: M:5644–5654). The §1 Apple table still calls Events recorded "Currently wrong — counts Razorpay". The *actual* current contamination, sandbox `ignored` events, is documented nowhere.
2. **Decoder §1, "`x% success` … denominator includes CREATED/FAILED attempts" is false.** PaymentStatus has no CREATED (`schema.prisma:672–678`), and a FAILED row only exists when a row was already there (PS:968–985). The rate is ≈100% by construction.
3. **Decoder §1 calls Paying Subscribers "the honest revenue-side number".** It includes MANUAL comps, sandbox testers (sandbox verify grants premium, AP:565–590) and internal accounts.
4. **Decoder §1 says every pre-2026-06-16 PAID order counts toward "need reconcile" forever.** The portal's v2.32.0 changelog (portal `CLAUDE.md`) measured 0 such rows on 2026-07-29. Q10c settles it. A brief that assumes the inflation (audit context item 3) should not.
5. **`SME_APPLE_IAP_API.md` §3 field reference lists `result` as `processed`/`orphan`/`error`.** The code also writes `ignored` for foreign-environment (sandbox) notifications (AP:874–878).
6. **`SME_APPLE_IAP_API.md` §4 says "Sandbox orders are hidden by default"** (orders and transactions only). It is silent on events. Sandbox events are *not* hidden, and sandbox users' `user_auth` premium state is *not* hidden either.
7. **Decoder §1 "Revenue (captured)" says nothing about refunds**, yet nothing ever sets a payment to `REFUNDED`. The figure is gross.
8. **Decoder §1 "Revenue by source" says a payment whose order wasn't fetched lands in "Unknown".** Since v2.20 such a payment falls back to `method=='apple_iap'` → APPLE (M:5713–5717), so only non-Apple orphans land in Unknown.
9. **Audit §1.2 (`AUDIT_2026-11.md:58`) lists the source as "`GET /sme/webhooks?source=APPLE`".** Correct on the wire, but `source` is sent via `params`, not on the path (M:5644–5646). No defect.
10. **Brief: "`premium` = row-level isPremium which includes trials (M:5658)" is confirmed.** But the dashboards label that number "conversion" (OPS:79, AD:30, CD:106), which neither the brief nor the decoder records.
11. **Audit header says backend `main` = `eafb96b`; current HEAD is `4ea16f2`.** No change to this data path except the additive `uiLanguage` (`8b26a50`).

## E. Processes and files

- No processes started; none to stop. No SQL run, no network calls, no edits to either repo.
- File written: `/private/tmp/claude-502/-Users-yashwanth-Desktop-projects/b0942abe-5468-460e-92b6-db352e6a3606/scratchpad/phase1/S01_analytics.md`.
