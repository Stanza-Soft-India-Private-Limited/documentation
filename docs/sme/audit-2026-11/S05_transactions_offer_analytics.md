# S05: `/transactions` (Payments · Orders · Webhooks) and `/offers/:id/analytics` (Phase 1 verifier)

Read-only pass, 2026-11-03, about 14:35–15:05 IST.

- Portal: `7b00daf` (v2.44.0). Backend `main`: `cca9ede`. Live evidence: prod v2.44.0, captured after the 14:13 IST deploy.
- Prod SQL was **SELECT only** (explicitly allowed by this lane's brief, which overrides the shared "no SQL" line). It ran through `scripts/staging-db.ts --prod`, and output was filtered with `rg -v 'eyJ|rzp_|sk-'`.
- Ids are cut to 8 characters. No names, emails or phones appear in this report.

Path keys:
- `TS:` = `admin-ui/src/pages/Transactions.tsx`
- `OA:` = `admin-ui/src/pages/OfferAnalytics.tsx`
- `API:` = `admin-ui/src/services/api.ts`
- `M:` = `app/main.py` (portal)
- `TX:` = `backend/src/modules/sme/services/sme-transaction.service.ts`
- `TXD:` = `…/dto/sme-transaction.dto.ts`
- `OF:` = `…/services/sme-offer.service.ts`
- `AP:` = `backend/src/modules/payments/services/apple-payments.service.ts`
- `PS:` = `…/payments/payments.service.ts`
- `RU:` = `…/sme/services/api-usage-rollup.service.ts`

SQL ids (T1…T7, O1…O3) refer to §E.

## A.0 Data path, verified

| Hop | Payments | Orders | Webhooks | Campaign analytics |
|---|---|---|---|---|
| UI call | `getSmePayments` API:2599 → `/sme/transactions?page=1&limit=20` (live: `transactions_network.txt` #211) | `getSmeOrders` API:2607 (#215) | `getSmeWebhooks` API:2615 (#216) | `getSmeOfferAbandoned` ×2 (`signal=all`, `signal=view`, `minAppVersion=1.8`, `limit=100`) + `getSmeOfferPurchased` (`offer_analytics_network.txt` #293–295) |
| Portal route | M:5390 → `_sme_proxy` M:5411 (10 params) | M:5438 → M:5458 | M:5485 → M:5502 (page, limit, source, dates, search; no exam) | M:7487 → M:7536; M:7539 → M:7573 |
| Our route | `sme-transaction.controller.ts:13` → TX:81 | :32 → TX:155 | :53 → TX:225 | `sme-offer.controller.ts:55` → OF:461; :71 → OF:740 |

- **Bug class 1 (params replacing the path query): not present.** Every caller passes `params=` with no query on the path. `_sme_proxy` drops `None` values and merges any path query anyway (M:4352–4366).
- **Bug classes 2, 3, 7: not on this screen.** No `_sme_fetch_all`. `/performance` uses it but the UI never calls it (§C).
- **No summary cards exist on `/transactions`.** The brief speaks of "summary cards". The only numbers on the three tabs are the pagination totals ("Showing 1–20 of 67 / 126 / 33430", "of 4 / 7 / 1672" pages; TS:297–306, 437–446, 525–534) plus the table columns. They are reconciled below as numbers.
- **The Webhooks tab did NOT render 0 rows.** `transactions_webhooks_snapshot.yml` and `_full.png` show 20 rows ("Showing 1–20 of 33430"), every one of them `payment.downtime.started|resolved · Razorpay · processed`. The brief's premise is contradicted (§D1). The real defect is the opposite: 99.5 % of the ledger is noise (W1).

## A. TRUST: per-number / per-column tables

Plane is **C** (billing tables) for all of `/transactions`. Window: **all-time** unless a date is picked. TZ: rows render in the **browser's local zone** (`toLocaleString('en-IN')` with no `timeZone`, TS:45–52). That is IST on an Indian laptop, but the page does not pin it.

Default exclusions on `/transactions`:
- `isTest` is excluded (TX:69–71, 93, 162).
- Internal accounts, MANUAL comps and Apple sandbox **events** are not excluded.

### A.1 Payments tab (UI default: no filters; T1, T1b, T7)

| # | Number / column (UI) | Our definition | Their transformation | Exclusions | Verdict + SQL result | Sev |
|---|---|---|---|---|---|---|
| P1 | **"Showing 1–20 of 67"** (TS:297–306) | `payment.count(where)` TX:128, `order.isTest=false` TX:93 | passthrough | isTest ✓ · internal ✗ · MANUAL ✗ | **TRUE as a row count, MISLABELLED as "payments".** SQL T1: 67 = 36 MANUAL (₹46,168 notional) + 20 Apple (₹27,283) + 11 Razorpay (₹22,392). Without internal accounts: 45 rows. **Rows that are real money from real people: 30, ₹43,775** (Apple 19 / ₹21,383 + Razorpay 11 / ₹22,392). So 54 % of "payments" are staff comps, and 22 of the 67 rows are internal accounts (21 comps + one internal Apple purchase of ₹5,900). | 🟠 |
| P2 | **Payment** id (TS:263) | `razorpayPaymentId ‖ appleTransactionId ‖ id` | — | — | **TRUE, RAW.** For a MANUAL comp it prints our internal UUID, which reads like a gateway id. | ⚪ |
| P3 | **User** (TS:250–271) | `order.user` (TX:123) | link to `/app-users/:id` | internal shown | TRUE. On page 1, 9 of the 20 rows are internal-domain accounts (live snapshot). | 🟡 |
| P4 | **Exam** (TS:273) | `order.examId` (TX:327) | null → "—" with a "not UPSC" tooltip (TS:80–90) | — | **TRUE.** T7: 0 payments have a null exam; 49 UPSC, 18 APPSC. | ⚪ |
| P5 | **Amount** `₹{pay.amount}` (TS:280) | `payments.amount/100` (TX:307) | none (no thousands separator: "₹5900") | MANUAL ✗ | **MISLABELLED for MANUAL.** A comp's amount is the exam's list price stamped as the default for the grant (S03: PS:1561–1571). No money moved, but it renders identically to a real ₹599 UPI charge. Non-INR: T2 found 0 non-INR orders, so the tooltip (TS:276–278) is correct but unused. | 🟠 |
| P6 | **Method** (TS:282) | `payments.method` | CSS `capitalize` | — | **TRUE.** T1: manual 36 / apple_iap 20 / upi 10 / card 1, the same as `/analytics`. | ⚪ |
| P7 | **Status** + status filter (TS:41, 210–215, 285) | `payments.status`. FAILED only *updates* an existing row (S01). | 5-option select | — | **CAPTURED on all 67 rows TRUE as stored. The filter is MISLABELLED.** T1b: 71 / 71 payment rows ever written are CAPTURED. PENDING, AUTHORIZED, FAILED and REFUNDED have **never** existed in prod, so 4 of the 5 options always return the "No payments found" empty state. Real failures exist only as **18 Razorpay `payment.failed` webhooks**, none attributable (T3b, T5b). Refunds table: 0 rows (T1b). A comp shown as "CAPTURED" is also mislabelled. | 🟠 |
| P8 | **Sandbox chip** + "Include sandbox" (TS:130–145, 286) | `order.isTest` | `includeTest` → `"true"` (TXD:117) | — | **TRUE.** 4 sandbox payments, all on internal accounts (T1b, T7). | ⚪ |
| P9 | **Date** (TS:289) | `payments.created_at` (not `captured_at`) | browser-local | — | TRUE. Not TZ-pinned. | ⚪ |
| P10 | **Date-range filter** (TS:223–226) | `new Date('YYYY-MM-DD')` = **UTC midnight**; `endDate` is `lte` that midnight (TX:73–79) | the UI sends the bare `<input type=date>` value | — | **WRONG (bug class 5).** (a) The start boundary is 05:30 IST, not 00:00. (b) `endDate` stops at 05:30 IST of that day, so the **end day is effectively excluded**. (c) A single-day range (start = end) is a zero-width instant and **always returns 0**. T4: 3 Nov returns **0** payments via our filter vs **2** in the true IST day. October orders: 24 vs 25 true. **Applies to all three tabs** (webhooks: 0 vs **219**). | 🔴 |
| P11 | Search "razorpay id / email" (TS:206) | `razorpayPaymentId` / `payments.email` / `user.email` (TX:96–105) | — | — | **TRUE to the placeholder.** But Apple transaction ids, our payment id and order id are not searchable, while TXD:102 says "Search payment/order id". | 🟡 |

### A.2 Orders tab (UI default; T2, T7)

| # | Number / column | Our definition | Transformation | Exclusions | Verdict + SQL result | Sev |
|---|---|---|---|---|---|---|
| O1 | **"Showing 1–20 of 126"** (TS:437–446) | `order.count` (TX:184), `isTest=false` | — | isTest ✓ · internal ✗ · MANUAL ✗ | **TRUE.** T2: PAID 67 / CREATED 57 / FAILED 2, the same as `/analytics` order status. Internal: 53 of 126 (real 73). PAID by source: APPLE 20, MANUAL 36, RAZORPAY 11. | 🟡 |
| O2 | **Status CREATED** = "abandoned checkout" (TS:418) | `orders.status` | badge | — | **WRONG for its support meaning.** T2 + extra query: the last CREATED order is **2026-08-22**, and 0 have been created since. Razorpay moved to UPI AutoPay **mandates**, which create a `subscriptions` row and **no order** until the first charge (S03: RS:255–270, 737–758). T5: **79 mandates since 2026-08-14, 71 never charged** (52 PENDING, 16 CANCELLED, 3 EXPIRED; 64 on non-internal accounts). On this tab, abandonment has looked like zero for 73 days. | 🔴 |
| O3 | **Amount** (`OrderAmount`, TS:162–184) | `amount/100`, `listAmount/100` (TX:365, 370) | strike-through list price when `listAmount ≠ amount` | — | **WRONG for Apple APPSC.** All **6** PAID Apple APPSC orders store `amount ₹499, list ₹599`. The UI draws ₹499 with ₹599 struck through, a ₹100 "discount" that never existed. Cause: AP:312–316 takes `listPrice` from the global `appConfig.monthlyPrice` (UPSC's ₹599) whatever the exam. MANUAL comps show the list price as if charged (see P5). | 🟠 |
| O4 | **Source / Plan / Exam** (TS:412–415) | columns | — | — | **TRUE.** T7: exam null = 0. | ⚪ |
| O5 | `reconcileHint` chips + the permanent amber explainer (TS:356–366, 419–420) | TX:344–351 | — | — | **TRUE.** T2: GRANT_PENDING 0, UNRESOLVED_EXAM 0. The amber box is standing copy for an empty state. | ⚪ |
| O6 | Offer-code chip (TS:422–426) | `orders.offer_code` | — | — | **TRUE.** 22 orders carry a code (21 CREATED, 1 FAILED), and 0 of them are PAID. | ⚪ |
| O7 | **Order** id column + search "order id / email" (TS:331, 402) | search = `razorpayOrderId` / `receipt` / `user.email` (TX:165–171) | — | — | **MISLABELLED.** The column prints our UUID, and pasting it into the "order id" search finds nothing. `orders.id` is not searched, despite TXD:102 saying it is. | 🟡 |
| O8 | Sandbox chip / toggle | as P8 | — | — | TRUE. 4 sandbox orders, all internal. | ⚪ |
| O9 | Date + date filter | as P9 / P10 | — | — | **WRONG**, as P10. | 🔴 |

### A.3 Webhooks tab (UI default: provider All; T3, T3b, T3c, T6)

| # | Number / column | Our definition | Transformation | Exclusions | Verdict + SQL result | Sev |
|---|---|---|---|---|---|---|
| W1 | **"Showing 1–20 of 33430"** (TS:525–534) | `payment_events WHERE kind='webhook'` (TX:227, 255) | — | none (`result` not filtered, TX:227–246) | **TRUE as a count (33,432 at query time), RAW/NOISE as a screen.** T3b: **33,276 (99.5 %) are Razorpay `payment.downtime.started/resolved`**, gateway status notices about no user and no payment (11,469 in the last 30 days, about 380 a day). T3c: rows 1–45 are all downtime. The first event about money is row 46 (`subscription.cancelled`). Meaningful rows: 156 in total. | 🟠 |
| W2 | **Result badge** (TS:545–552, 566–576) | `payment_events.result` | `processed` → green, tooltip "Applied"; `orphan` / `error` styled; **no case for `ignored`** | — | **MISLABELLED.** (a) `processed` on a downtime notice means nothing was applied, yet the tooltip says "Applied". (b) T6: **22 Apple sandbox rows are `ignored`** (last on 08-18) and render as a grey chip with **no tooltip**, because `error` is null for all 22. (c) **6 more sandbox notifications (06-24 to 06-29) are stored `processed`**: they predate the environment gate (AP:870–879) and cannot be told apart on screen. Decoded JWS: Production 32 / Sandbox 28 of 60 Apple rows. | 🟡 |
| W3 | **Exam** chip + copy "The exam shown on an event is read off the order it matched … A blank user on an account-level event is normal" (TS:496–500, 577) | `examId` from `orderId` (TX:263–291) | ExamBadge only when `examId` is set | — | **WRONG copy.** T3: **0 of 33,432 rows carry `order_id` or `user_id`.** Razorpay (PS:819–825) and Apple (AP:848–854) record before matching and never backfill (S03). So the exam chip **never** renders, and a blank user is universal, not an account-level special case. | 🟠 |
| W4 | Provider chips (TS:474–490, 539–543) | `source` → provider (TX:229–236) | — | — | TRUE. Apple 60 / Razorpay 33,372. | ⚪ |
| W5 | Search "event id / type" (TS:470) | `externalId` / `type` (TX:241–245) | — | — | TRUE to the placeholder. It cannot find a user, an email, a subscription id or a payment id: the payload is not searched. | 🟡 |
| W6 | Expanded row = `JSON.stringify(evt)` (TS:518–520) | `payload` = raw body (TX:280) | — | — | **RAW.** Apple `payload` is a JWS string, so the sandbox/production environment is not visible without decoding. | 🟡 |
| W7 | Date filter | `receivedAt`, as P10 | — | — | **WRONG**: T4 gives 0 vs 219 for 3 Nov IST. | 🔴 |

### A.4 Support rows: "what a support person needs"

| Case | What the three tabs show today | What support needs to say | Where it already persists |
|---|---|---|---|
| **Razorpay mandate attempt** (e.g. S03 user 3a5a9b8f: 3 mandates, never charged) | **Payments: nothing** (no payment until a charge). **Orders: nothing** (a mandate creates no order; O2). **Webhooks:** `subscription.cancelled` / `.pending` / `.authenticated` rows exist (T5b: 18 / 3 / 5) but carry no user or order (T3), and search reaches only event id or type. The person cannot be found on `/transactions` at all. | "She tried UPSC Monthly AutoPay on 24 Sep, 2 Nov and 3 Nov. None was authorised. Two were cancelled and one is still pending." | `subscriptions` (`user_id`, `provider`, `status`, `plan_type`, `exam_id`, `created_at`, `first_charge_granted_at`, `external_id`). Webhook rows can be joined by parsing `raw_body` (Razorpay `payload.subscription.entity.id` = `subscriptions.external_id`). Not done today; no new persistence needed. |
| **Apple sandbox event** | Payments and Orders: hidden by default; the toggle shows 4 sandbox orders (all internal). **Webhooks:** `apple.DID_RENEW · Apple · ignored`, grey, no tooltip, no user, raw JWS. Six pre-gate sandbox events say `processed`. | "This was a TestFlight/sandbox renewal by a team tester. Apple sent it to production and we ignored it on purpose. No money, no premium." | `payment_events.result='ignored'` (post-gate). The environment is in `raw_body` (JWS `data.environment`). The tester can be recovered from `apple_subscriptions` / `orders.apple_original_transaction_id` via the decoded `originalTransactionId`. |

### A.5 Campaign analytics `/offers/62b20aac/analytics` (FREEDOM15)

Campaign facts (O1):
- Window 2026-08-09 18:30Z → 2026-08-23 18:29Z (10–23 Aug IST), `upsc-cse`.
- Status **LIVE**, although the window closed 71 days ago.

Every funnel number comes from `abandoned.funnel` (OF:673–725). Internal accounts are not excluded upstream.

| # | Card / number (UI) | Plane | Our definition | Portal transformation | Window / TZ | Exclusions | Verdict + SQL result | Sev |
|---|---|---|---|---|---|---|---|---|
| C1 | Header **LIVE** badge + "showing 10 Aug 2026 – 23 Aug 2026" (OA:699–710, 685–689) | C | `campaign.status`; `window` echoed after clamping (OF:651–653) | IST dates | campaign window | — | **MISLABELLED (minor).** LIVE while the window closed 71 days ago. `/offers` says "Live, window closed"; this page does not. | 🟡 |
| C2 | Verdict sentence **"₹0 from 0 buyers. 0 people opened the offer, 10 reached checkout, none paid."** (OA:806–815) | A/B/C mix | `revenuePaise`, `paid`, `viewed`, `joined` | words | window | none | **₹0 / 0 buyers TRUE. "0 people opened" WRONG (UNKNOWABLE).** VIEW reads raw `api_usage` (OF:541–548, 689–693), purged after 30 days (RU:30, 120–123). O2: oldest `api_usage` row is 2026-10-03, and there are **0 `offer:%` rows of any campaign in the table**. The 20–23 Aug views were captured and then deleted. **"10 reached checkout" is MISLABELLED**: O1 / O3 show 6 of the 10 are internal accounts; **4 are customers**. | 🔴 |
| C3 | **Revenue ₹0** · "0 Apple · 0 Razorpay" (OA:820–823) | C | sum PAID campaign-coded orders (OF:705–706) | paise → ₹ | window | is_test ✓ | **TRUE.** T2: 0 PAID orders carry any offer code. | ⚪ |
| C4 | **Buyers 0** (OA:825–828) | C | distinct PAID users (OF:697) | — | window | is_test ✓ | **TRUE.** | ⚪ |
| C5 | **Discount given ₹0** · "**₹17,119 was offered and never claimed**" (OA:830–836) | C | `discountUnclaimedPaise` = Σ(list − amount) over **every non-PAID campaign order** (OF:707–709) | words | window | is_test ✓; internal ✗; per **order** | **₹0 TRUE. ₹17,119 MISLABELLED.** O1: 19 orders × ₹901, so one person's repeat attempts count again (per distinct user it is ₹9,010). 14 of the 19 orders belong to internal accounts. **Customers were offered ₹3,604** (4 users × ₹901). | 🟠 |
| C6 | Funnel **Viewed 0** + "since 20 Aug" badge, tooltip "Capture began 20 Aug 2026 … any window opening earlier under-reports" (OA:404–411, 456–463); amber note (OA:867–875); limits panel "Views before 20 Aug 2026 do not exist" (OA:1280–1282) | B | as C2 | — | window | — | **WRONG.** The badge gives the wrong reason. Capture *did* cover 20–23 Aug of this window; the rows are gone because of the **30-day purge**. Every campaign older than 30 days will read Viewed 0 permanently, and the page attributes that to the August deploy. The daily rollup cannot recover it: it is keyed by route, not by `feature` (`schema.prisma` ApiUsageDaily:764–778; O2: 441 paywall-route rows / 376 users in the window, not campaign-specific). | 🔴 |
| C7 | **Reached checkout 10** · "19 orders from these users" (OA:413–420) | C | distinct users with campaign-coded non-test orders, any status (OF:696, 710) | — | window | is_test ✓ · internal ✗ | **TRUE to the definition, MISLABELLED as demand.** O1: joined 10 / real 4; orders 19 (customers own 5). | 🟠 |
| C8 | **Paid 0** (OA:421–429) | C | OF:697 | — | window | — | TRUE. | ⚪ |
| C9 | Drops: "no drop recorded" (viewed → joined) and "**−10 abandoned**" (OA:433–446) | — | `viewedNotJoined`; `joined − paid` | — | — | — | **"no drop recorded" WRONG** (an artefact of the purge, C6). **"−10 abandoned" MISLABELLED**: 6 internal; 4 of the 10 were later comped (C11). | 🟠 |
| C10 | Signal chips **Reached checkout 5 · Checkout + tap 1 · Opened the offer 0 · iOS upgrade tap 12** (OA:899–919, count at 901) | A (orders) / auth_events / B | rows from `all` + `view`, deduped per user to the furthest stage (OA:337–373); counted **before** the internal filter | local | window | exclusions OF:605–614; internal ✗ in counts | **ORDER 5 + BOTH 1 TRUE** (O3: 6 non-excluded = 3 internal + 3 customers). **Opened 0 WRONG** (C6). **TAP 12 TRUE today, decaying.** O2: 64 Apple tap events from 15 users on build ≥1.8 in the window, before exclusions. `auth_events` is purged at 90 days (RU:32, 128–131), so this window's taps disappear **between 2026-11-10 and 2026-11-21**, and the chip will fall to 0 with no notice. TAP is also **not exam-scoped**: any iOS upgrade tap in the window, any exam (OF:517–537). | 🟠 |
| C11 | Bridge **"10 reached checkout · 4 excluded (already converted) · 3 internal hidden · 3 listed"** (OA:939–962; `rulesExcluded` OA:635–639) | C | exclusion: premium active now **or** any PAID order after campaign start (OF:605–614) | `joined − ORDER/BOTH rows` | now | — | **"4 excluded (already converted)" MISLABELLED.** O3: all 4 were dropped because of later **MANUAL comps** (3 of the 4 are internal). Nobody converted by paying. "3 internal hidden" TRUE (here the portal's 2-domain rule and the audit's 4-pattern rule agree, O3). "3 listed" TRUE (3 customers). The rule matches `SME_CAMPAIGN_ABANDONED_API.md:198` ("by any route"); the *label* is what misleads. | 🟠 |
| C12 | Chase table columns (OA:996–1098) | mixed | **Attempts** = orders + taps (+ views only if neither) (OF:558–561); **Platform** = the platform on the campaign view, else the user's **latest `api_usage` platform ever** (OF:595–601); **Order** = latest order amount/list; **Last seen** = `lastAttemptAt` | `platformLabel(null)` = "Unknown" | window / now | — | **Attempts: mixed units** (one order plus three taps = 4). **Platform MISLABELLED**: 2 of 3 rows show "Unknown" because their `api_usage` rows were purged, not because the platform is unknown. **Last seen MISLABELLED**: it is the last campaign attempt (Aug), not last seen in the app. **Order ₹4,999 / ₹5,900 TRUE.** One listed customer (f75f65b2) also has a Razorpay mandate (O3) that the page cannot show. | 🟡 |
| C13 | Band 4 empty state "Nobody bought on this campaign" (OA:1179–1193) | C | `/purchased` = PAID with offer code (OF:740–830) | — | window | is_test ✓ | **TRUE** (0 rows). | ⚪ |
| C14 | iOS floor select (OA:1118–1129); "floor" banner (OA:765–773) | — | `minAppVersion`, 10×100 bound | — | — | — | TRUE. | ⚪ |

**`/offers` list (new items only, beyond the C2 ledger):**
- The card's "**Link only + banner** — the dashboard banner still advertises it to every non-paying user" is shown on a campaign whose window closed 71 days ago. The banner route requires the window, so that tooltip is stale for this state. UNVERIFIED against the live `/paywall/banner`; do not act on it without checking.
- "window closed 71d 14h ago" with status LIVE: campaigns never auto-end. Not a number defect.

## A.2 Exam scoping

| Tab / card | Exam axis | Effect |
|---|---|---|
| Payments | parent `orders.exam_id` (TX:89–95); `none` = null | Correct. Prod: 0 null exams (T7), so "Unresolved exam only" is always empty today. The **Method / Status / Source** filters are not exam-blind; they compose. |
| Orders | `orders.exam_id` (TX:161) | Correct. Same null = 0. **But O3:** Apple APPSC `listAmount` uses UPSC's price, so per-exam "discount" is wrong for APPSC. |
| Webhooks | **none**, by design (M:5485–5513 forwards no exam; TX:258–262) | The UI says the exam is read off the matched order; **no event has ever matched an order** (W3), so every row is exam-less. Cross-exam by construction. |
| Mandates | `subscriptions.exam_id` exists | Not on any tab (A.4). |
| Campaign funnel / ORDER / VIEW | campaign `offer_code` (one campaign = one exam, `campaign.examId` OA:795–797) | Correctly exam-bound. |
| Campaign **TAP** | **none**: any Apple `upgrade_tapped` / `checkout_opened` in the window (OF:517–537) | An APPSC iOS tap lands in a UPSC campaign's list. Labelled "inferred", but not "any exam". |
| Chase-list exclusion "premium now" | person-level `user_auth.premium_expires_at` (OF:608) | A user comped in APPSC is "already converted" for a UPSC campaign. |

## B. MEANING

> Owner ruling (2026-11-03), quoted: "The numbers are there BUT those numbers are not explaining anything — same on the aspirant profile screen. No need to drastically change stuff; compact the details or combine and show. I don't need to see the endpoints a user is hitting on the portal, it explains nothing. Instead explain in simple terms: some X metric done on this data, or 'he stayed X mins', or 'he has been using this many times'. I may be grossly oversimplifying — the portal might already do some things well and corrections may not be needed at all. Just analyze and judge."

Six-question coverage:

| Q | This screen |
|---|---|
| (1) Feature use + repeat rate | no (not its job) |
| (2) Per-exam / per-language funnels | **Partially.** The exam filter works on Payments and Orders. The campaign page is per-exam, apart from TAP. No language axis anywhere. Language persists (`user_profiles.ui_language`); not needed here. |
| (3) Where buyers stop in checkout | **Not answered on `/transactions`**: no CREATED orders since 08-22 (O2), and mandates are invisible. **Partially answered on campaign analytics** (orders + taps), but the view stage has been purged and internal accounts dominate. Data persists: `subscriptions`, `orders` CREATED/FAILED, `auth_events` checkout_opened / abandoned / purchase_failed (90 d). |
| (4) Question quality | n/a |
| (5) Notification outcomes | n/a (the chase-list notify is a sender, not an outcome) |
| (6) Support view of one user incl. Razorpay mandates | **Not answered.** Mandates are absent (A.4). Webhooks cannot be tied to a user. Comps look like payments. Data persists (`subscriptions`, `orders.payment_source='MANUAL'`, `sme_audit_log` PREMIUM_GRANT); the webhook→user link needs `raw_body` parsing (no new persistence). |

### B.1 Card by card: plain-language test and ruling

| Card / table | Plain sentence a non-engineer can say today | Ruling | Derived metric to show instead + computable today? |
|---|---|---|---|
| Payments list + "of 67" | "67 payments", which is false: 36 are staff comps and 22 rows are our own team. **Fails.** | **COMPACT/COMBINE** (keep the table, add one sentence above it, relabel comps) | **"This month: ₹X received from N customers (Apple ₹a · Razorpay ₹b); M staff comps given, not money; refunds ₹0."** = Σ `payments.amount` WHERE `status='CAPTURED'` AND `orders.payment_source<>'MANUAL'` AND NOT `orders.is_test` AND user not internal, bucketed by IST month of `payments.created_at` (or `captured_at`), minus Σ `refunds.amount` joined on `refunds.payment_id` (0 rows today). Comps = count of `orders` with `payment_source='MANUAL'`, plus `sme_audit_log` PREMIUM_GRANT. **Computable today.** All-time check: ₹43,775 from 30 real payments. In the table, render MANUAL rows as "Staff comp (no payment)" instead of "₹599 · manual · CAPTURED". |
| Payment status filter | No sentence: 4 of its 5 values can never match. **Fails.** | **COMPACT** | Drop the dead options, or replace them with "Failed payment attempts (gateway)" = Razorpay `payment_events.type='payment.failed'` count per IST day (18 all-time). **Computable today**, but tying them to a user needs `raw_body` parsing: `payload.payment.entity.email/contact/notes` → user. No new persistence. |
| Orders list + "of 126" | "126 orders: 67 paid, 57 abandoned." True numbers, false meaning: the 57 are all before 22 Aug, and 71 uncharged mandates since then are missing. **Fails for (3)/(6).** | **COMPACT/COMBINE** | **"Attempts vs paid, per gateway, this month."** Razorpay attempts = `subscriptions.created_at` in month + `orders` CREATED/FAILED (RAZORPAY); Razorpay paid = `subscriptions.first_charge_granted_at` in month + `orders` PAID RAZORPAY. Apple attempts = `auth_events` `checkout_opened` with `metadata.gateway='apple'` (90-day cap); Apple paid = `orders` PAID APPLE. Example sentence: "Razorpay: 9 people started AutoPay this month, 2 got charged." **Computable today** (Apple attempts only within 90 days). Add mandate rows to the Orders list as their own status ("AutoPay set up, not charged"), read from `subscriptions`. Needs a backend read (no SME route reads `subscriptions`); the data persists. |
| Orders: amount strike-through | "APPSC buyers got ₹100 off": false (O3). | **KEEP** the display, **fix the data source** | `listAmount` for Apple must come from the order's exam plan, not `appConfig.monthlyPrice` (AP:312–316). Backend fix, flagged only. |
| `reconcileHint` chips + amber box | "No paid order is waiting for premium." True, but never said: the box is permanent. | **COMPACT** | One line: "0 paid orders need a manual grant" from the count of `reconcileHint≠OK` (`orders` PAID AND `premium_granted_at` IS NULL). Show the long explainer only when the count is above 0. **Computable today** (T2 = 0). |
| Webhooks list (33,430) | No sentence. 99.5 % is Razorpay gateway-downtime chatter. **Fails: RAW/NOISE**, exactly the "endpoints explain nothing" complaint. | **RAW/NOISE → derive** | **"Payment providers: last real event 3 min ago; this week 4 captured, 2 failed, 1 AutoPay cancelled, 0 refunds; Razorpay reported downtime 14 times today."** = `payment_events` grouped by `type` family and IST day, with `payment.downtime.*` collapsed to a single count, plus `max(received_at)` per provider over the non-downtime types. Hide `downtime.*` and `result='ignored'` by default behind a toggle (needs a `type`/`result` filter param, which our DTO lacks; flag). **Computable today.** Per-user attribution: parse `raw_body`, no new persistence. |
| Webhook result badge | "processed" reads as "applied" and isn't; "ignored" means nothing to the reader. | **COMPACT (relabel)** | `ignored` → "Sandbox test, ignored on purpose"; downtime → "Gateway status notice". Data: `result`, `type`. **Computable today.** |
| Campaign verdict sentence | "₹0 from 0 buyers; 10 reached checkout; none paid." **Passes the form, fails on truth**: 6 of the 10 are staff, and "0 opened" is deleted data. | **KEEP the form, fix the inputs** | "₹0 from 0 buyers. **4 customers** (plus 6 staff accounts) reached checkout; none paid. **Views not available: usage older than 30 days is deleted.**" Customer counts = `funnel.joined` minus internal users (portal rule or upstream; the funnel is server-side, so this needs an upstream internal exclusion or a `joinedExcludingInternal` field; flag). The VIEW retention note is computable today (`now() - window.from > 30 d`). |
| Revenue / Buyers / Discount tiles | "₹0, 0 buyers, ₹0 discount given": passes. "₹17,119 was offered": fails (one person counted 4 times, mostly staff). | **KEEP** Revenue and Buyers; **COMPACT** the discount sub-line | "₹901 off was offered to 4 customers; nobody took it" = per distinct non-internal user, `max(list_amount − amount)` over non-PAID campaign orders. **Computable today** (O1 variant). |
| Funnel (Viewed → Reached checkout → Paid) | Viewed fails (purged). The rest passes once staff are separated. | **COMPACT** | Hide or grey Viewed when the window started more than 30 days ago, and state why. Long-term VIEW history **needs new persistence** (e.g. `feature` carried into the daily rollup); flag only. |
| Signal chips + bridge sentence | "3 customers to chase": TRUE and useful. "4 excluded (already converted)": false. | **KEEP** (best-designed part of the screen); **relabel** one clause | "4 excluded: **later given premium by staff**" vs "**paid later**", split by `payment_source` of the PAID order after start (O3 query). **Computable today.** TAP: add "available until 21 Nov (90-day retention)", computed from `max(created_at)+90d`. |
| Chase table columns | "1 attempt, Unknown platform, last seen 17 Aug": half true. | **COMPACT (relabel)** | "Last tried" (not "Last seen"); Platform from `orders` (the Razorpay order is web/Android) or "not recorded (usage expired)"; Attempts split into "2 orders · 3 iOS taps". **Computable today** from the same CTEs. |
| Who bought (empty) | "Nobody bought on FREEDOM15", which is true and said plainly. | **KEEP AS IS** | — |

### B.2 Every COMPACT/COMBINE proposal reshaped to the `/insights/purchase-funnel` template

The owner ruled `/insights/purchase-funnel` the house template ("very excellent"; SHARED_RULES "B reference"; live: `insights_purchase_funnel_snapshot.yml`). Every proposal below copies its anatomy:
1. a one-line subtitle;
2. a scope line;
3. step tiles with plain counts, and the loss between steps labelled with its likely cause;
4. one headline ratio with its caveat;
5. failures split into **the customer's (pricing / friction)** and **ours to fix**;
6. one per-day chart;
7. the standard controls: **exam selector + 7/14/30/60/90 d window pills**.

It copies the shape, not the chart.

Example numbers are **real prod values for the last 30 IST days** (since 2026-10-04 00:00 IST), all exams, internal accounts excluded (query B1 in §E). They show the sentences are fillable today, not mock-ups.

Minimal-change rule: these replace the *header* of each existing tab or band. The existing tables stay underneath as the drill-down.

#### B.2.1 `/transactions` → Payments tab header: "Money received"

| Template part | Proposal | Source (table.column → derivation) | Today? |
|---|---|---|---|
| Subtitle | "What actually reached our account: real payments by gateway, with staff comps and refunds kept separate. Comps are not money." | — | — |
| Scope line | "Showing the last 30 days for all exams · team accounts and App Store sandbox excluded" | window = `payments.created_at` IST day; exam = `orders.exam_id`; exclusions = internal rule + `orders.is_test` | ✓ |
| Step tiles | **Paid by customers ₹6,089 (11 payments)** → split: Apple ₹3,793 (7) · Razorpay ₹2,296 (4). Beside them, never added: **Staff comps 6 to 5 customers** ("₹3,294 list value, no money", + 11 to team accounts, hidden). **Refunds ₹0.** | `payments.amount` WHERE `status='CAPTURED'` ⋈ `orders` (`payment_source<>'MANUAL'`, NOT `is_test`, user not internal); comps = `orders.payment_source='MANUAL'`; refunds = Σ `refunds.amount` by `refunds.created_at` | ✓ |
| Headline ratio + caveat | **"Net received ₹6,089 = gross ₹6,089 − refunds ₹0"**. Caveat: "Apple amounts are what Apple charged in ₹ before its commission; a non-INR storefront is reported separately, not converted." | `orders.storefront_currency` / `charged_amount_minor` for the non-INR split | ✓ |
| Customer's vs ours | Not a funnel screen. Instead, one amber line **only when non-zero**: "**N paid orders are waiting for premium (ours to fix)**", from `reconcileHint≠OK`. This replaces the permanent amber explainer (O5). Today 0, so hidden. | `orders` PAID AND `premium_granted_at` IS NULL | ✓ |
| Per-day chart | Stacked bars per IST day: Apple ₹ / Razorpay ₹; comps as a separate dotted marker, never stacked into ₹ | as above, bucketed `(created_at AT TIME ZONE 'UTC') AT TIME ZONE 'Asia/Kolkata'` | ✓ |
| Controls | exam selector (exists, TS:222) + window pills (new; they replace the broken date inputs, P10) | the pills must send IST instants (`…T00:00:00+05:30`), the same fix OA:165–166 already uses | ✓ (UI only) |

#### B.2.2 `/transactions` → Orders tab header: "Who tried to pay, and where they stopped" (per gateway)

This is the screen that answers shareholder questions (3) and (6). It is a gateway-specific extension of the purchase funnel: it starts at **checkout**, where the purchase funnel's server truth stops, and adds the mandate step that no screen shows today.

| Template part | Proposal | Source | Today? |
|---|---|---|---|
| Subtitle | "Where paying customers stop: checkout opened → payment set up → money taken, per gateway and IST day. The losses between steps are the point." | — | — |
| Scope line | "Showing the last 30 days for all exams · team accounts excluded · Apple steps come from app 2.0+ only (a floor)" | as purchase-funnel | ✓ |
| Step tiles: **Razorpay (UPI AutoPay / card)** | **Checkout opened 23** → (−?) → **AutoPay started 25 people (36 attempts)** → **−21 · 84 % lost · "approval not completed in the UPI app"** → **Charged 4** → **Paid ₹2,296** | opened = `auth_events.event_type='checkout_opened'` non-Apple (distinct users, 90 d cap); started = `subscriptions` (provider RAZORPAY, `created_at`, distinct `user_id`); charged = `first_charge_granted_at IS NOT NULL`; legacy one-time orders = `orders` CREATED/FAILED (0 since 22 Aug) | ✓ |
| Step tiles: **Apple** | **Checkout opened 24** → **−17 · 71 % lost · "closed the App Store sheet: price or second thoughts"** → **Paid 7 · ₹3,793** | `auth_events` `checkout_opened` with `metadata.gateway='apple'`; `orders` PAID APPLE | ✓ (90 d cap) |
| Caveat on the step counts | "Opened" is client-emitted (app 2.0+), while "AutoPay started" is server truth, so opened can be **lower** than started (23 < 25 here). Same rule as the purchase funnel's "conversion can exceed 100 %": steps are floors, not a strict chain. | — | — |
| Headline ratio + caveat | **"Razorpay: 4 of 25 people who started AutoPay were charged (16 %)."** Caveat: "A PENDING mandate can still be approved later; a rate for the last few days will rise. Pending ≠ lost until 24 h old." The 24 h sweep→ABANDONED is a pending proposal in memory; until then, age is computed from `subscriptions.created_at`. | `subscriptions` | ✓ |
| **Customer's (pricing / friction)** | **Checkout abandoned 38 people** (`auth_events.checkout_abandoned`) · **AutoPay not approved: 21 pending, 11 cancelled** (`subscriptions.status`, uncharged) · **Bank / UPI declined: 10** (`payment_events` `payment.failed`; count only, no person attached) | `auth_events`, `subscriptions`, `payment_events.type` | ✓ (the declines are unattributable to a person without `raw_body` parsing) |
| **Ours to fix** | **Purchase failed in the app: 4 people** (`auth_events.purchase_failed`) · **App refused a retry ("already active") — N taps** (`api_usage` `POST /api/v1/payments/subscriptions` status 400, 30 d cap; S03 E10; fixed in `eafb96b`, so this tile should read 0 after 2.2) · **Paid but premium not granted: 0** (`reconcileHint`) | `auth_events`, `api_usage` (30 d), `orders` | ✓ (400 reason not persisted, S03 M11) |
| Per-day chart | One chart per gateway (or one with a gateway toggle): opened / started / charged bars + conversion line; a gap = no denominator that day | IST day buckets of the sources above | ✓ |
| Controls | exam selector (`subscriptions.exam_id`, `orders.exam_id`; Apple taps carry an exam only on 2.0+ `auth_events.metadata`, otherwise "all exams") + window pills | — | ✓, with the TAP-exam caveat (A.2) |
| Drill-down (existing table) | Keep the Orders table, but add mandate rows read from `subscriptions` ("AutoPay set up · not charged · 2 Nov"), and render MANUAL as "Staff comp (no payment)" | needs a backend read of `subscriptions` (no SME route today) | data ✓, route ✗ (flag) |

#### B.2.3 `/transactions` → Webhooks tab header: "Are the payment providers talking to us?" (RAW → derived)

| Template part | Proposal | Source | Today? |
|---|---|---|---|
| Subtitle | "Whether Apple and Razorpay are reaching us, and what they told us about money. Gateway status chatter is counted, not listed." | — | — |
| Scope line | "Showing the last 30 days · events cannot be tied to an exam or a person yet" (true today, W3) | — | ✓ |
| Step tiles (one row per provider, not a funnel) | **Razorpay:** last money event 2 h ago · captured 4 · failed 10 · AutoPay charged 4 / cancelled 17 · *downtime notices 11,722 (collapsed)*. **Apple:** last event 6 h ago · new / renewed / expired / refunds · *sandbox ignored N (collapsed)* | `payment_events` grouped by `provider`, `type` family, `result`; `max(received_at)` over non-downtime types | ✓ |
| Headline ratio + caveat | **"73 money events vs 11,722 status notices in 30 days."** Caveat: "A silent provider (no money event for more than 24 h while orders are being paid) means webhooks are not landing. Check health before trusting any status here." | as above + `orders` PAID in the same window | ✓ |
| Customer's vs ours | Customer's: `payment.failed` (bank / UPI declined), `subscription.cancelled`. Ours: `result='error'` rows, and "paid order with no provider event within 1 h" (`orders` PAID vs `payment_events` by `raw_body` id) | `payment_events.result`, `type`; the join needs `raw_body` parsing | ✓ / parse |
| Per-day chart | Money events per IST day by type family; downtime as a faint background band | `payment_events` | ✓ |
| Controls | window pills + provider chips (exist). The exam selector is **deliberately absent** (no exam on events) and the scope line says so. | needs `type` / `result` filter params on `/sme/webhooks` to hide noise server-side | flag (DTO has none) |

#### B.2.4 `/offers/:id/analytics`: already the closest to the template; align it

The page already has a subtitle-like verdict, step tiles with drops, and a who-to-chase list. Changes to match the reference, minimal:

| Template part | Today | Proposal | Source | Today? |
|---|---|---|---|---|
| Subtitle | verdict sentence (OA:806–815) | keep; replace "0 people opened" with "views unavailable (usage older than 30 days is deleted)" when `now − window.from > 30 d` | `window.from` | ✓ |
| Scope line | "showing 10 Aug – 23 Aug" (OA:708) | "Showing 10–23 Aug · UPSC · **4 customers + 6 team accounts**" (team counted, not hidden) | needs `funnel` to report internal separately (upstream field) or a client re-count from rows; the funnel is server-side (OA:70–73) | flag |
| Step tiles + labelled losses | Viewed → Reached checkout → Paid with "−10 abandoned" (OA:433–446) | **Reached checkout 4 (+6 team)** → **−4 · 100 % lost · "price — campaign ₹4,999 vs nothing; 1 later comped by staff"** → **Paid 0**. Viewed is greyed with the retention note instead of reading "0". | `orders.offer_code`; comp split from O3 (`payment_source` of the PAID order after start) | ✓ |
| Headline ratio + caveat | none explicit | **"Paid ÷ reached checkout: 0 of 4 customers."** Caveat: "iOS buyers who closed the App Store sheet leave no order; they appear only as unattributed taps (expires after 90 days)." | `funnel.paid / joined` (excluding internal) | ✓ |
| Customer's vs ours | the "4 excluded (already converted)" clause | Customer's: "3 left at checkout (price / second thoughts)". Ours / not demand: "4 later given premium by staff", "6 team accounts testing". Never "converted". | O3 logic | ✓ |
| Per-day chart | **absent** | **add one**: campaign orders created / paid per IST day across the window (+ iOS taps as a faint inferred series while retained) | `orders.offer_code`, `created_at`; `auth_events` (90 d) | ✓ |
| Controls | Promoted-from / Until date inputs (OA:726–763) | keep the date inputs: a campaign has its own window, so 7–90 d pills do not fit here; this is the one deliberate deviation. The exam is fixed by the campaign (shown as a chip). | — | — |

#### B.3 Drill-down: every proposed count, and whether we can list the users behind it today

Owner ruling (SHARED_RULES, "Drill-down ruling"): every step or tile count must open a sidebar listing the people behind it. Our `/sme/analytics/purchase-funnel` returns **counts only** (`sme-purchase-funnel.service.ts:125–131`, `count(DISTINCT user_id)`), so it cannot feed any sidebar.

| Count (proposal) | Source table.column | Retention | User ids returnable today? |
|---|---|---|---|
| Paid by customers / per gateway (B.2.1, B.2.2 "Paid") | `payments` ⋈ `orders."userId"`, `payment_source` | permanent | **Yes**: `GET /sme/transactions?source=&exam=&startDate=&endDate=` and `GET /sme/orders?orderStatus=PAID&source=` carry `userId`. The internal exclusion is client-side. The date params must be IST instants (P10 bug). |
| Staff comps | `orders.payment_source='MANUAL'` | permanent | **Yes**: `GET /sme/orders?source=MANUAL&orderStatus=PAID` |
| Refunds | `refunds.user_id` | permanent | **No SME route** → new endpoint (flag) |
| Paid but not granted (`reconcileHint`) | `orders.premium_granted_at` | permanent | **Partly**: `GET /sme/orders?orderStatus=PAID` then filter on `reconcileHint` locally (no hint param); `?exam=none` gives the UNRESOLVED subset. |
| Checkout opened (Apple / Razorpay) | `auth_events.user_id`, `event_type='checkout_opened'`, `metadata.gateway` | **90 d** | **No** (counts only) → new per-step user-list endpoint (flag) |
| AutoPay started / charged / pending / cancelled | `subscriptions.user_id`, `status`, `first_charge_granted_at`, `exam_id` | permanent | **No SME route reads `subscriptions`** → new endpoint (flag) |
| Checkout abandoned / Purchase failed | `auth_events.user_id`, `event_type` | **90 d** | **No** → new per-step endpoint (flag). (A per-user trail exists, `/sme/users/:id/activity`, but no list by step.) |
| Bank / UPI declined (`payment.failed`) | `payment_events` | permanent | **No, and not attributable**: `user_id` / `order_id` are null on 100 % of rows (T3). Needs a match/backfill at record time (new persistence, flag), or read-time `raw_body` parsing (`payload.payment.entity.email/contact/notes`). |
| "Already active" refusals | `api_usage.user_id`, route, status 400 | **30 d** | **No** list route (per-user incidents only, S03) → new endpoint (flag) |
| Webhook money events / downtime (B.2.3) | `payment_events.type` | permanent | **Rows yes** (`GET /sme/webhooks?source=&search=<type>`), **people no** (unattributed, as above) |
| Campaign: reached checkout / ORDER / BOTH / TAP / VIEW | `orders.offer_code`; `auth_events`; `api_usage.feature` | ORDER permanent · TAP 90 d · VIEW 30 d | **Partly**: `GET /sme/offers/:id/abandoned` returns `userId` per row, but **only after exclusions**. The 4 "excluded (later comped)" people inside `funnel.joined` cannot be listed → needs an `includeExcluded` (or `reason`) param (flag). VIEW is unrecoverable after 30 d. |
| Campaign: paid | `orders.offer_code` PAID | permanent | **Yes**: `GET /sme/offers/:id/purchased` |
| Campaign: team accounts (scope line) | `user_auth.email` | permanent | **Yes for listed rows** (email on each `/abandoned` row); no for excluded ones (as above) |

**Not reshaped (KEEP AS IS, per B.1):** Who bought (Band 4), the chase-list table (relabel-only changes in B.1), and `reconcileHint` (folded into B.2.1 as a conditional line).

## C. GAP

| Item | Source | State |
|---|---|---|
| `GET /sme/transactions/:id` (payment detail incl. `notes` = `rawJws`, storefront) | `SME_PORTAL_API.md` §2, `SME_APPLE_IAP_API.md` §2.1; TX:140–153 | **proxied-only** (M:5429; `getSmePayment` API:2603 unused) |
| `GET /sme/orders/:id` | TX:196–206 | **proxied-only** (M:5476; `getSmeOrder` API:2611 unused) |
| `appleOriginalTransactionId` ("group renewals by it, visible in detail") | `SME_APPLE_IAP_API.md` §2.2/§4 | **absent on our side**: column exists (`orders.apple_original_transaction_id`), `formatOrder` TX:353–388 never emits it. Re-confirmed. |
| Order `payments[]`, `premiumGrantedAt`, `razorpayOrderId`, `appleTransactionId`, `receipt`, `currency`, `updatedAt` | TX:355–386 | proxied-only (in the TS type, API:1946–1972; not rendered). `razorpayOrderId` is the only order id that search accepts (O7). |
| Payment `contact`, `vpa`, `bank`, `wallet`, `errorCode`, `errorReason`, `capturedAt` | TX:312–318 | proxied-only. `contact`/`vpa`/`bank`/`wallet`/`errorCode`/`errorReason` are not in the TS type at all (API:1916–1942). |
| Webhook `result='ignored'` meaning | AP:870–879; not in `SME_APPLE_IAP_API.md` §3 field list (S01 D5) | rendered as a bare chip, unexplained (W2) |
| Webhook `error` | TX:286 | rendered only when `result='error'` (TS:579); 0 such rows in prod |
| Webhook filter by `result` / `type` family | — | **absent both sides**: no DTO param, so the downtime noise cannot be hidden server-side |
| `?exam=` on webhooks | docs: deliberately ignored | correct absence |
| `userId` filter (payments, orders) | TXD:51–54; forwarded M:5397, 5444 | proxied-only: the UI never sends it; `/app-users/:id` does not deep-link here with `?userId=` |
| `/sme/offers/{id}/performance` | M:7608–7648 | proxied-only, unused by design (OA:75–78). Its `startDate` scan is read as UTC midnight (TX:76): the M:7627–7631 note "re-check … if read as UTC" is **confirmed true**, and the first IST day's 00:00–05:30 is missing from the scan. Moot while unused. |
| `/abandoned` `signal` (re-labels) | `SME_CAMPAIGN_ABANDONED_API.md` §2 | rendered correctly via two fetches + dedupe (OA:337–373) |
| `/abandoned` `from` / `to` | doc | rendered (OA:726–763) |
| `/abandoned` `pageCounts` | doc | unused as a headline (correct) |
| `funnel.paidByPlatform`, `discountGivenPaise`, `discountUnclaimedPaise`, `orderCount`, `viewedNotJoined` | OF:713–723 | rendered |
| `/purchased` `pageTotals` | OF:815 | proxied-only (fine: page-local) |
| Mandates (`subscriptions`) on any money screen | — | **absent on our side** (no SME route reads `subscriptions`; S03) |
| Refunds | `refunds` table | **absent on our side** (no SME read); 0 rows in prod today |
| **Stale UI copy** | TS:496–500 ("read off the order it matched"; "blank user … normal") — false for 100 % of rows (W3) · TS:571 "Applied" tooltip on downtime notices · OA:458, 871–873, 1280–1282 ("since 20 Aug … cannot be backfilled") — the binding limit is the 30-day purge · OA:946 "already converted" (C11) · OA:1006 "Last seen" (C12) · TS:331 "order id / email" (O7) | stale / false |
| Portal docstring vs code | M:7502–7504 "There is deliberately NO date parameter … Do not add one." The same handler forwards `from`/`to` (M:7531–7535). | stale docstring |
| Portal calls to changed or deprecated routes | — | none |

## D. Contradictions

1. **Brief: "`transactions_webhooks_*` … rendered 0 rows — find out why."** Contradicted by the evidence. The snapshot (`transactions_webhooks_snapshot.yml`, the `f6e1110` group of 20 `payment.downtime.*` rows) and `transactions_webhooks_full.png` show 20 rows of 33,430. No filter default, query or `result` column hides anything. The finding is the reverse: an unfiltered ledger that is 99.5 % noise (W1).
2. **Brief: "Reconcile EVERY number on the three transactions tabs' summary cards."** No summary cards exist on `/transactions` (A.0). The numbers are the pagination totals and the columns, reconciled in A.1–A.3.
3. **`SME_PORTAL_API.md` §2.4 / `SME_APPLE_IAP_API.md` §3 and TX:258–262, 289–291 say `examId` comes from "the order the event was matched to", null only for unmatched events.** Prod: **0 / 33,432** webhook rows have `order_id` (T3). Events are recorded before matching and never backfilled (PS:819–825; AP:848–854). Matching never happens, so the null is universal and the UI copy built on the doc (TS:496–500) is false.
4. **`SME_CAMPAIGN_ABANDONED_API.md:35–38` and OF:670–671 say "Every other signal is complete for the full campaign history"; `SME_CAMPAIGN_ANALYTICS_PAGE_GUIDE.md:54` says VIEW is deploy-forward from 2026-08-20.** Retention, not the deploy date, is the binding limit:
   - VIEW = raw `api_usage`, 30 days (RU:30). O2: 0 `offer:%` rows remain for any campaign.
   - TAP = `auth_events`, 90 days (RU:32). FREEDOM15's taps expire 11-10 to 11-21.
   - Only ORDER is permanent.
   - The C2 ledger predicted this; it is now proven on prod data.
5. **`TXD:102` "Search payment/order id, receipt, or user email".** Code: payments search razorpay id + emails (TX:96–105); orders search razorpay order id + receipt + email (TX:165–171). Neither searches our `id`, which is what both tabs print in their id column.
6. **`SME_PORTAL_API.md:489` and the UI status select list PENDING / AUTHORIZED / FAILED / REFUNDED as payment statuses.** Prod has only ever stored CAPTURED (T1b: 71 / 71). Nothing sets REFUNDED (S01); FAILED only overwrites an existing row (S01).
7. **Apple `listAmount` is "the configured list price" (AP:279), and the portal draws it as the undiscounted price (TS:177–181).** It is the global `appConfig.monthlyPrice` / `annualPrice` (AP:312–316), not the exam's. All 6 Apple APPSC orders show a fake ₹599 list vs ₹499 charged.
8. **Portal docstring M:7502–7504 ("NO date parameter … Do not add one")** vs M:7531–7535, which forwards `from`/`to`.
9. **Portal comment M:7627–7631 (performance scan "assumes upstream reads `startDate` as an IST day")** is answered: it is read as UTC midnight (TX:76). Moot today (route unused).
10. **`/analytics` "Apple events recorded 60"** (from the brief's evidence): 60 = 32 Production + 28 Sandbox (22 `ignored` + 6 pre-gate `processed`) by decoded JWS (T6). This refines S01 #27, which said only "counts sandbox". Every other `/analytics` money card in the brief reconciles exactly:
    - Revenue ₹49,675 = Apple ₹27,283 + Razorpay ₹22,392.
    - 67 payments; MANUAL ₹46,168.
    - PAID 67 / CREATED 57 / FAILED 2.
    - manual 36 / apple_iap 20 / upi 10 / card 1.
    - Without internal accounts, the captured Apple + Razorpay figure is **₹43,775**.
11. **OA:87–90 comment "effectively all [certain abandoners] were team accounts"** still holds on a smaller scale: 6 of 10 joined are internal (O3), and the funnel counts them all.

## E. SQL run (production, read-only) with results

Runner: `cd backend && npx ts-node -T scripts/staging-db.ts --prod --file <f> | rg -v 'eyJ|rzp_|sk-'`. Files: `/private/tmp/claude-502/-Users-yashwanth-Desktop-projects/b0942abe-5468-460e-92b6-db352e6a3606/scratchpad/s05sql/{t1,t1b,t2,t3,t3b,t3c,t4,t5,t5b,t6,t7,o1,o2,o3}.sql`. Internal rule in every query: `email ILIKE '%@stanzasoft.%' OR '%@prepmonkey.%' OR '%@designasylum.in'`.

| Id | Query (shape) | Result |
|---|---|---|
| — | `information_schema.columns` for orders, payments, payment_events, subscriptions, refunds, offer_campaign, auth_events | Columns as the brief says (`orders."userId"`, `created_at`, `apple_original_transaction_id` present; `payment_events.user_id` / `order_id`; `subscriptions.first_charge_granted_at`) |
| T1 | payments ⋈ orders, NOT is_test, ROLLUP(source, status, method): n, n_real, ₹, ₹_real | ALL 67 / 45 real / ₹95,843 / ₹63,062 real. APPLE 20 / 19 / ₹27,283 / ₹21,383. MANUAL 36 / 15 / ₹46,168 / ₹19,287. RAZORPAY 11 / 11 / ₹22,392 (upi 10, card 1). All CAPTURED. |
| T1b | status universe, is_test, refunds, error fields, captured_at | 71 payments ever, **all CAPTURED**; is_test 4; refunds **0 rows**; error fields 0; captured_at never null on CAPTURED |
| T2 | orders NOT is_test, ROLLUP(status, source) with hints / offer / exam-null / non-INR / dates | ALL 126 (73 real, 53 users). CREATED 57 (all RAZORPAY, 27 real, 21 with offer code, **last 2026-08-22**). FAILED 2. PAID 67 (APPLE 20, MANUAL 36 / 15 real / 13 users, RAZORPAY 11). GRANT_PENDING 0, UNRESOLVED 0, exam null 0, non-INR 0. |
| T2x | CREATED orders after 2026-08-22 18:30Z; last-30-day and today counts | 0; 28; 2 |
| T3 | payment_events kind='webhook' ROLLUP(provider, result) with user / order counts | 33,432; apple 60 (processed 38, ignored 22); razorpay 33,372 processed; **with_user 0, with_order 0** everywhere |
| T3b | type families | `payment.downtime.*` **33,276** (11,469 in 30 d); payment.failed 18; subscription.cancelled 18; apple.DID_RENEW ignored 17 / processed 8; apple.SUBSCRIBED 15 + 1 ignored; payment.captured 10; order.paid 10; payment.authorized 10; settlement.processed 5; subscription.charged / authenticated / activated 5 each; invoice.paid 5; apple.EXPIRED 5 + 2 ignored; DID_FAIL_TO_RENEW 4; subscription.pending 3; DID_CHANGE_RENEWAL_STATUS 3; apple.TEST 3; OFFER_REDEEMED 2 ignored; halted 1; paused 1 |
| T3c | newest-first ranks | rows 1–20 (page 1) all downtime; first non-downtime row at rank **46** |
| T4 | the backend date predicate vs the IST day | payments 3 Nov: **0** vs 2. Orders 1–31 Oct: 24 vs 25. Webhooks 3 Nov: **0** vs 219. |
| T5 | subscriptions by provider / status / charged | RAZORPAY: PENDING uncharged 52 (49 real, 51 users, 08-14 → 11-03); ACTIVE charged 2; PAUSED charged 1; CANCELLED uncharged 16 (15 real, 9 users); CANCELLED charged 2; EXPIRED uncharged 3 (0 real) |
| T5b | Razorpay non-downtime webhooks with user / order counts | 11 types; every one has u = 0, o = 0 |
| T6 | Apple webhook rows by decoded JWS `data.environment` × result | Production processed 32; Sandbox **ignored 22** (06-30 → 08-18); Sandbox **processed 6** (06-24 → 06-29, pre-gate) |
| T7 | orders and payments by exam × is_test × status (n, real, real-money) | orders: APPSC PAID 18 / 10 / 7; UPSC CREATED 57 / 27 / 27, FAILED 2 / 1 / 1, PAID 49 / 35 / 23; UPSC sandbox PAID 4 / 0 / 0. Payments mirror the PAID rows. |
| T8 | PAID orders: source × exam × amount × list | **APPLE APPSC ₹499 list ₹599 ×6**; APPLE UPSC ₹599 list ₹599 ×6, list null ×5; ₹5,900 ×3; MANUAL APPSC ₹499 ×10, ₹4,999 ×1; MANUAL UPSC ₹599 ×21, ₹5,900 ×4; RAZORPAY UPSC ₹599 ×7, ₹5,900 ×3; RAZORPAY APPSC ₹499 ×1 |
| T9 | Apple rows: result × error | processed 38 / null; ignored 22 / **null** (so no tooltip) |
| O1 | offer_campaign + funnel recompute | FREEDOM15 (62b20aac) LIVE, 08-09 18:30Z → 08-23 18:29Z, upsc-cse: **joined 10, joined_real 4, orders 19, paid 0, unclaimed ₹17,119 per order / ₹9,010 per user**. INDE50 and QATEST: 1 joined each, 0 real. |
| O2 | VIEW / TAP evidence vs retention | `api_usage` `offer:FREEDOM15` rows **0**, any `offer:%` **0**; oldest api_usage **2026-10-03 20:01Z**; oldest auth_events 2026-08-04; TAP events in window 64, distinct users ≥1.8 **15**; purge dates for in-window taps **2026-11-10 → 2026-11-21**; api_usage_daily paywall route in window 441 rows / 376 users (not campaign-specific); oldest daily 2026-07-05 |
| O3 | the 10 joined users: internal by both rules, status, premium now, PAID after start, mandates | Internal (both rules agree): 45c3c221, 611f9073, 89c0603e, 9d6cbc67, a8bde6fb, e8255054. Excluded as "converted" = the 4 with **MANUAL** PAID after start: 51691e39 (customer, comp 09-26), 89c0603e, a8bde6fb, e8255054 (internal). Listed customers: 822c8563, dd352cdb, f75f65b2 (the last has 1 Razorpay mandate). |

| B1 | last 30 IST days, all exams, internal excluded (`s05sql/b1.sql`) | Razorpay AutoPay started 25 users / 36 mandates, charged 4 users; uncharged PENDING 21, CANCELLED 11. Razorpay webhooks in window: payment.failed 10, subscription.authenticated 4, charged 4, cancelled 17. `checkout_opened` users: Apple 24, Razorpay 23. checkout_abandoned 38 users, purchase_failed 4. Real paid: APPLE 7 / 7 / ₹3,793; RAZORPAY 4 / 4 / ₹2,296. Comps to customers 6 orders / 5 users / ₹3,294 notional (+11 to team). Team real-money orders 0. Webhooks: downtime 11,722 vs 73 other. Last non-downtime event: Apple 11-03 02:51Z, Razorpay 11-03 05:49Z. |

### Processes and files

- Processes started: one background `mkdir`/heredoc shell (task `bwq8b2jcc`) that hung after writing its files. It was stopped with TaskStop and never had a pid. All `ts-node` runs ran in the foreground under `timeout` and exited. No other processes remain.
- No edits to either repo, no git operations, no browser, no portal or API calls, no mutating SQL.
- Files written:
  - this report;
  - the SQL files in `/private/tmp/claude-502/-Users-yashwanth-Desktop-projects/b0942abe-5468-460e-92b6-db352e6a3606/scratchpad/s05sql/`.
