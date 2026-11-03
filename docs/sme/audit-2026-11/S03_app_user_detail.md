# S03 — `/app-users/:id` aspirant profile / support screen (Phase 1 verifier)

Paths:
- `UI:` = `portal/admin-ui/src/`. `AUD` = `pages/AppUserDetail.tsx`. `SH` = `components/features/app-users/SnapshotHeader.tsx`. `SV` = `…/SessionsView.tsx`. `SU` = `…/session-utils.ts`. `IP` = `…/IncidentsPanel.tsx`. `TL` = `…/Timeline.tsx`. `DU` = `…/detail-utils.tsx`. `UT` = `…/useTrail.ts`.
- `M:` = `portal/app/main.py`.
- `AS` = `backend/src/modules/sme/services/sme-activity.service.ts`. `US` = `backend/src/modules/sme/services/sme-user.service.ts`.
- `RS` = `backend/src/modules/payments/services/razorpay-subscriptions.service.ts`. `PS` = `backend/src/modules/payments/payments.service.ts`.

Versions and evidence:
- Backend: local HEAD `cca9ede`. The prod image is upsc:164, deployed 14:13 IST today.
- Portal: clone `7b00daf` (2026-10-27); live v2.44.0.
- Live captures from 2026-11-03, in `projects/.playwright-mcp/live/`:
  - `app_user_detail_mandate_{network.txt,snapshot.yml,full.png}`: user **3a5a9b8f**, non-internal.
  - `app_user_detail_paid_*`: user **ab74a3b2**, non-internal.
- Prod SQL: read-only, run by me through `scripts/staging-db.ts --prod`. I selected ids, counts and dates only.
- Names, emails and phones are redacted. Ids are truncated to 8 characters.

---

## Headline (read this first)

**A user who tried to pay and failed looks, on this page, like a user who never tried. The page shows this live, today.**

### Mandate user 3a5a9b8f

What the database holds:
- **3** Razorpay UPI AutoPay mandates, none charged:
  - `b16afd74` MONTHLY, created 24 Sep, now CANCELLED.
  - `df7c4046` MONTHLY, created 02 Nov 13:53 IST, now CANCELLED.
  - `4fbc284a` ANNUAL, created 03 Nov 11:19 IST, still PENDING.
- **28** `400` responses on `POST /payments/subscriptions`.
- **26** client `purchase_failed` events, plus `checkout_opened` / `checkout_abandoned` on 3 separate days.

What the page renders:
- "No purchases on any exam. A trial creates no row, so this is normal for a trial user" (`SH:305`).
- "**No orders on this account.**" (`SH:370`).
- "0 Orders" and "0 Payment webhooks".

Why:
- The snapshot reads **`orders` only** (`AS:288–297`, `take: 5`).
- No SME read path touches the **`subscriptions`** table. `rg "subscription\." src/modules/sme` finds no Prisma read of it.
- A mandate creates a `subscriptions` row and **no order** (`RS:255–270`). An order is created only when a charge succeeds (`RS:737–758`).
- The only trace on the page is the Incidents panel: "46 failures on POST /api/v1/payments/subscriptions" and "8 failures". That is a route name and a status code, not the sentence "she tried to buy UPSC Monthly 3 times and could not".

### Scale in production (non-internal users)

- **47** users hold **61** never-charged Razorpay mandates (46 PENDING, 15 CANCELLED).
- **42** of those 47 users have **zero** orders, so their profile says "No orders on this account".

### What the 46 / 8 / 4 "failures" were

They are **not 46 failures**:
- Each tap is counted twice: one `api_usage` 400 plus one client `purchase_failed`.
  - 3a5a9b8f on 2 Nov: 24 × 400 + 22 × `purchase_failed` = 46.
  - 3a5a9b8f on 3 Nov: 4 + 4 = 8.
  - ab74a3b2 on 2 Nov: 2 + 2 = 4.
- Every 400 follows a `200` that created a PENDING row, within ≤10 minutes:

| User | 200 that created the PENDING row (IST) | 400s that followed (IST) |
|---|---|---|
| 3a5a9b8f | 13:53:32 | 13:54:07 → 14:03:12 |
| 3a5a9b8f | 11:19:44 | 11:19:49 → 11:19:54 |
| ab74a3b2 | 14:04:52 | 14:05:27 |

That pattern is the pre-`eafb96b` rule:
- `ABANDONED_PENDING_MS = 15 min` → `'You already have an active subscription. Cancel it before starting another.'`. Source: `git show eafb96b^:…razorpay-subscriptions.service.ts` L38, L172, L328.
- `eafb96b` (2026-10-31) is in HEAD, but it only reached prod with today's 14:13 IST deploy. All 34 of these 400s predate it.

Ruling out the other `400` causes in `RS`:
- `:141` already premium: neither user held a live paid entitlement on those dates.
- `:220` campaign code: `offer_code` is null on every row.
- `:912` subscriptions disabled, and `:973` plan not configured: both ruled out, because the 200 a few seconds earlier succeeded.

⚠️ This is **inferred, not proven**. `api_usage` stores no response body and `purchase_failed` carries no `error_code`/`message` (both null in prod), so the exact message is unrecoverable.

### Paid user ab74a3b2

The coordinator described this user as a "paid Razorpay subscriber". **He is not one.** Per prod SQL:

| Period | Orders | Money |
|---|---|---|
| Apr–Aug | 4 × Apple Production MONTHLY ₹599 | real |
| 16 Sep, 17 Sep, 03 Nov (×2) | **4 × MANUAL SME comps**, `PREMIUM_GRANT` in `sme_audit_log` | none |
| 2 Nov 14:04 IST | Razorpay mandate `68ba0b23`, still PENDING, never charged | none |

The two orders "8 seconds apart" on 3 Nov 10:55:54 (UPSC ₹599) and 10:56:02 (APPSC ₹499) are **two manual grants, one per exam**. They are not a double charge or a reprice, and no money moved.

The page renders them as "**PAID** ₹599.00 … MONTHLY · MANUAL". The amount is the exam's list price, used as the default for a comp (`PS:1561–1571`). In production that is **36 MANUAL "PAID" orders for 13 users, ₹46,168 notional** that a support person can read as received money.

---

## A. TRUST

### A.0 Data path (verified, plus the live network capture)

**What fires on page open.** The live capture (mandate, lines 171–179) shows 8 calls:
- `snapshot?days=30`
- `GET /sme/users/:id`. The detail call fires from `AUD:134–139` and is **not listed in AUDIT §1.2** (see §D).
- `incidents?days=30&gapMinutes=5&limit=20`
- Timeline × 3 streams (`auth_events`, `api_usage`, the other 15 sources at `limit=132`), via `UT:34–38` and `UT:45–47`
- One `api_usage` page-2 cursor
- `timeline?sources=notes&limit=50` (NotesComposer)

**Proxies.** The proxy routes at M:4635, M:7002, M:7012, M:7033, M:7058 and M:7069:
- forward every param the UI sends;
- validate `days` 1–90 and `limit` 1–200 / 1–50;
- drop `None` (M:4352–4355).

**Bug classes:**
- **Class 1 does not apply.** No path carries a query string, and `_sme_proxy` merges anyway (M:4356–4366).
- **Classes 2, 3 and 7 do not apply** on this page: no `_sme_fetch_all`, no local filtering, explicit `Query()` params.
- **Class 4 (internal accounts):** the profile never flags a staff or test account (no internal badge).
- **Class 5:** the UI formats everything through `Asia/Kolkata` (`DU:47–76`). Our day keys are IST (`AS` `istClock`, `getISTDateString`). The `days` window is a rolling `now − days×24h` (`AS:220–222`, `AS:1640–1662`), not IST-day aligned.
- **Class 6:** the 90d preset is offered (`TL:26`), and `retention.note` renders in all four places (`SH:683–691`, `TL:312–316`, `SV:264–268`, `IP:176–180`).
  - Prod: the oldest raw `api_usage` row is 2026-10-03 20:01 UTC, so the purge **is running**, at 30 days.

**Mutations.** The page's only write is **notes**. Grant, revoke, trial extension, notify, deactivate, delete and edit live on the `/app-users` drawer (`AppUsers.tsx:1094–1124`). Their guards:

| Action | Our guard | Portal role | UI confirmation |
|---|---|---|---|
| Grant | exam validated `US:323–326`, MANUAL order + audit `US:327–345` | `require_admin_or_app_admin`, examId required M:4673–4685 | modal form `AppUsers.tsx:225–368` |
| Revoke | `?exam` required, else 400 `US:362–406` | M:4688 | modal plus a **second** confirm for `all` `AppUsers.tsx:1736–1801` |
| Trial extension | blocked for paying / suspended `US:432–445` | M:6466 | modal |
| Notify | — | M:4776 | modal |
| Deactivate | reversible, audited `US:270–293` | — | **one click, no confirm** (`AppUsers.tsx:1331`, `:1676`) 🟡 |
| Delete | — | `require_admin` M:4664 | type-the-email confirm `AppUsers.tsx:915–958` |
| Notes | 1–2000 / 60 / 120 chars (dto `:209–225`); a 5xx is surfaced, not swallowed (`AS:1427–1451`) | — | — |

There is no per-action role on our side: one shared key. Author is self-typed (labelled as such).

### A.1 Per-number table

UI values are from the live captures. SQL values are prod results for the two users: **M** = 3a5a9b8f, **P** = ab74a3b2.

| # | Number as labelled (UI file:line) | Plane | Our definition | Their transformation | Window + TZ | Exclusions | Verdict (UI vs SQL) | Reconcile SQL (E#) | Sev |
|---|---|---|---|---|---|---|---|---|---|
| 1 | **"Recent orders (last 5)" / "No orders on this account."** (`SH:367–376`) | C | `orders WHERE userId ORDER BY createdAt DESC LIMIT 5` (`AS:288–297`); no `subscriptions`, no payments | renders status / exam / plan·source / amount | all time | none (MANUAL and isTest included) | **WRONG for the support question.** M: UI "No orders" ✔ vs SQL 0 orders ✔, but **3 mandates exist** (E2). Population: 42 users say "No orders" while holding uncharged mandates (E14). | E1, E2, E14 | 🔴 |
| 2 | Recent-order row "**PAID ₹599.00 · MONTHLY · MANUAL**" (`SH:189–236`) | C | `orders.amount/100`, `payment_source` | `money()` | — | MANUAL not distinguished | **MISLABELLED.** P: UI shows 4 MANUAL "PAID" rows (₹599, ₹599, ₹599, ₹499). SQL: all MANUAL comps, payment `manual`, no money. Prod-wide: 36 rows, 13 users, ₹46,168 notional (E15). | E3, E15 | 🟠 |
| 3 | Recent-order "test" chip (`SH:206–213`) | C | `GET :id` emits `isTest` (`US:218`); **snapshot select omits it** (`AS:288–297`; DTO `:427–438`) | reads `order.isTest` from the **snapshot** | — | — | **WRONG (dead chip).** It can never render. Prod impact today is small: 4 sandbox orders, 1 user, internal (E16). | E16 | 🟡 |
| 4 | "What they hold · per exam" cards: exam / LIVE·EXPIRED·REVOKED / source / until·perpetual / "trial still covers" (`SH:255–314`) | C | `user_exam_entitlements` → `describeEntitlements` (`entitlement.service.ts:173–213`) | `accessTier` in tooltip only (`SH:260`); `isTrial` rendered (`SH:282–286`) | live | — | **TRUE.** P: UI APPSC LIVE MANUAL to 03 Dec and UPSC LIVE MANUAL to 03 Dec = SQL (E4). M: UI none = SQL none. | E4 | ⚪ |
| 5 | Empty-holdings copy "No purchases… normal for a trial user" (`SH:299–306`) | C | `trialEndsAt` is sent **only while in trial** (`AS:359`) | the "trial ended {date}" branch (`SH:304`) is therefore dead | — | — | **MISLABELLED.** M is "Trial Ended" (trial over since ~May) and attempted to buy 3×, yet the copy says "normal for a trial user". | E1 | 🟠 |
| 6 | Account-wide pill "Trial Ended / Premium", tooltip "isPremium: true — entitlement active" (`SH:328–335`) | C | `derivePremiumState` / `isPremiumUser` incl. trial (`premium-check.util.ts:117–129`, `:302–325`) | tooltip says "entitlement active" | person-level | — | **TRUE** pill (M "Trial Ended", P "Premium" = SQL status). **MISLABELLED** tooltip: `isPremium` is also true for an in-window trialist. | E1 | 🟡 |
| 7 | "Premium expires", "Trial", "Trial length from" (`SH:346–364`) | C | `premium_expires_at`; `trialEndsAt`/`trialDaysLeft` only in trial; `trialExamId` from `GET :id` | — | — | — | **TRUE.** P: "1mo from now" = 2026-12-03 05:26 UTC. M: "—", "not in trial". | E1 | ⚪ |
| 8 | **Today (IST) quota** "AI Mentor chat 0/3 (3 left, daily)", "content doc 1/subject lifetime", footer "**resets at midnight IST — no history is kept**" (`SH:410–453`) | Redis + Postgres | `QuotaService.getMe(userId, activeExamId‖upsc-cse)` (`quota.service.ts:84–132`); `pyq_open` lifetime from Postgres (`free-question-set.service.ts:204–211`) | footer applied to every row | today IST | exemption includes trial | **MISLABELLED** for `lifetime` and `lifetime_per_subject` rows (footer and "Today" title are false for them). "Premium ·" prefix (`SH:445`) also shows for a trial user (exemption). Values not reconcilable (Redis TTL). | — | 🟡 |
| 9 | **Streak** "0 days · best 0" (M) / "2 days · best 2 · last 2026-09-17" (P) (`SH:455–473`) | Neo4j | `userRepo.getUserById` (`AS:544–574`). **Postgres `user_auth.current_streak/max_streak/last_streak_date` has been authoritative since `3c8bc41` (2026-08-26)**; Neo4j is a fire-and-forget shadow (`streak.service.ts:210–240`, `schema.prisma:91–99`) | — | IST day | — | **TRUE today for M and P** (E5: 0/0/null; 2/2/2026-09-17), but it reads the wrong store. Also "**2 days**" for P is a streak broken 47 days ago: `current_streak` is never reset, and the app does the same (`streak.service.ts:317–343`). → **MISLABELLED** as "current". | E5 | 🟠 |
| 10 | "Last login" 2mo ago (M) (`SH:480–486`) | C | `user_auth.lastLoginAt` (token issuance) | RelTime | — | — | **TRUE** (2026-08-25 05:41 UTC). Meaningless to support. | E1 | ⚪ |
| 11 | "**Last active**" 3h (M), 4h (P) | C | `last_active_at`, stamped **once per IST day on first app open** (`wylto-sync.service.ts:200–218`, `schema.prisma:80–84`) | — | — | — | **MISLABELLED.** It is "first opened today", not last active. P: Last active 4h ago while Last session is 1h ago. | E1, E6 | 🟡 |
| 12 | "Last session" 3h / 1h | C | `max(sessions.lastAccessedAt)`, updated on **every** authenticated request (`jwt.strategy.ts:57–60`) | — | full history | — | **TRUE** (M 05:49:53 UTC, P 07:43 UTC). This is the true "last opened the app" and should be the one clock shown. | E6 | ⚪ |
| 13 | "Last request" + raw route "POST /api/v1/payments/subscriptions" (`SH:522–530`) | B | newest `api_usage` row (`AS:242–247`) | route printed verbatim | 30d purge → null for idle >30d | — | **TRUE but RAW.** The owner's "endpoints explain nothing". | — | 🟡 |
| 14 | "Last client event" + `purchase_failed` | auth_events | newest `auth_events` (`AS:249–254`) | raw event type | 90d | server- and client-written mixed (`metadata.source`) | TRUE, RAW | — | ⚪ |
| 15 | App versions "2.1 · android", "2.0 · android", "1 device" (`SH:537–551`) | B | distinct (`app_version`, `platform`) / `deviceId` in `api_usage` in window (`AS:255–270`) | "pre-1.7" for null | window, but ≤30d effective | — | TRUE (P: 3 devices). An idle >30d user reads "No app version recorded". | — | ⚪ |
| 16 | Push tokens "android · granted" (`SH:553–561`) | C | `device_tokens` (`AS:272–284`) | title `istFull(lastUsedAt)`; `lastUsedAt` is nullable → "01 Jan 1970" on null (`SH:504`) | — | — | TRUE; tooltip bug on null | — | ⚪ |
| 17 | "Platforms android" (`SH:148–155`) | C | distinct `platform` of **active push tokens** (`US:170–173`, `:202–204`) | label "Platforms" | — | — | **MISLABELLED** ("devices that can receive push", not "platforms used") | — | ⚪ |
| 18 | "Onboarded Yes", "Role", "Provider google", "Status UNSUBSCRIBED lifecycle", "Member since" (`SH:109–158`) | C | `GET :id` profile / user_auth | `onboardingCompleted: null` → "No" (`SH:141–143`) | — | — | TRUE (null-as-No edge only) | E1 | ⚪ |
| 19 | **Time in app · last 30d · estimate**: Sessions "≥2–3", Total "≥1–12 min", Median "<1–1 min", Active days "≥1–3" (M); "≥2", "≥14–33 min", "7–17 min", "≥1" (P) (`SH:586–635`) | B + auth_events | client-side `estimateTimeInApp` over the **budgeted partial read**: 2 pages × 3 streams (`UT:40`, `UT:171–178`); gap = 30 min (`SU:26`, `SU:370–385`); pairs (`SU:333–362`) | "≥" floor and an amber "Partial read — since {date}" banner | 30d nominal; real floor = 30 Oct (M), **3 Nov** (P) | none | **WRONG by default** (honestly floored, but headline-wrong). Server-side full 30 days (E7): **M = 8 sessions / 15 min / 7 IST days; P = 34 sessions / 147 min / 21 days.** The page shows 2 / 33 min / 1 for P until "Load full 30 days" is pressed. | E7 | 🟠 |
| 20 | "**Activity · 708 events**" (M) / "2144 events" (P) (`SH:650–652`) | mixed | sum of 16 per-source counts (`AS:398–404`) | — | 30d rolling | — | **RAW / NOISE.** Sums requests + pushes we sent + staff actions; 593 of M's 708 are requests. | E8 | 🟡 |
| 21 | Count tiles: "593 Requests", "83 Auth & client", "0 Orders", "0 Payment webhooks", "0 Question attempts", "32 Notifications" … (`SH:657–682`) | mixed | `activityCounts` (`AS:427–503`): 16 sources; **no `payments` count, no `subscriptions`, no `pyq_attempts`, no `mains_evaluations`, no `daily_sessions`** | `SOURCE_META` labels (`DU:167–185`) | 30d | — | Counts **TRUE** as table counts (M: Orders 0 = SQL; Notifications 32 = SQL 32). But "**Payment webhooks**" is **WRONG as labelled**: in prod **0 of 33,370 Razorpay and 0 of 60 Apple webhook rows carry `user_id` or `order_id`** (E9). Only 14 `client_verify` rows ever match a user (`PS:819–826`, `apple-payments.service.ts:848–854`, `PS:276–290` never backfills). "Question attempts" = content-doc MCQs first answered (`createdAt`), **not PYQs**. | E8, E9 | 🔴 (webhooks) / 🟡 |
| 22 | P: "**2 Orders**" tile beside a 5-row "Recent orders" list | C | tile = orders created in the 30-day window (`AS:461`); list = last 5 all-time (`AS:288–297`) | — | two scopes on one card | — | **TRUE each, MISLEADING together.** SQL: 2 orders in 30d (both 3 Nov MANUAL); 8 all-time. Neither says "2 comps, 0 real payments in 30 days". | E3 | 🟡 |
| 23 | **Incidents** "46 failures on POST /api/v1/payments/subscriptions (+1 other) · 400 ×24 · purchase_failed ×22" (`IP:56–69`) | B + auth_events | `api_usage.status >= 400` ∪ `auth_events` in `FAILURE_EVENT_TYPES` (`AS:1210–1235`, `:1384–1392`), clustered at a 5-minute gap | route and status chips | 30d | `?exam` narrows the api half only | **TRUE as counts, MISLABELLED as meaning.** Double-counts each tap. A 402 paywall counts as a "failure". The true reason (false "already active" block) is not recoverable. | E10, E11 | 🟠 |
| 24 | Incidents empty state "**No failures… Nothing broke for this user — you can say so.**" (`IP:200–208`) | B | as above. **Excludes** `orders.FAILED`, `payments.FAILED`, `payment_events.error`, `checkout_abandoned`, abandoned mandates, and guard-thrown 401/402/403 (never captured, S02 A.0) | — | 30d | — | **WRONG.** A user whose UPI mandate was abandoned, or whose Razorpay payment FAILED via webhook, gets "nothing broke — you can say so". | E2, E12 | 🔴 |
| 25 | **Sessions view** row "App opened · about 6 min · android 2.1 · **258 requests** · 23 errors · 22 sign-in/client errors · UPSC ×227 APPSC ×31 · purchase_failed · upgrade_tapped … +51 more" (`SV:66–186`) | B + narrative | `buildSessions` (`SU:246–285`) over the partial read | — | window; floor shown | — | Durations **TRUE** (pairs or gap, labelled est.). "258 requests" and per-exam request counts are **RAW**. "22 sign-in/client errors" is **MISLABELLED**: they are `purchase_failed`, not sign-in. Copy "no PYQ-attempt source upstream" (`SV:129`, `SU:4–8`) is **stale**: `pyq_attempts` has existed since 2026-08-26. | E13 | 🟠 |
| 26 | Notification rows "Notification '…' — read/unread" | A | `notification_history.isRead` (`AS:1053–1072`) | — | — | — | TRUE row-wise. "read" = bulk feed open (S02). Per-push `notification_dispatch.tapped_at` not shown: M 32 sent / 0 tapped, P 59 / 0 (E13). | E13 | 🟡 |
| 27 | Exam chip "UPSC" (activeExamId), quota exam chip, focus bar | C | `user_profiles.active_exam_id` | null → "No exam chosen" | — | — | TRUE | — | ⚪ |
| 28 | Notes list | C | `sme_audit_log` SUPPORT_NOTE | — | — | — | TRUE | — | ⚪ |

**Numbers the brief asked for that the page does not show at all** (verdict: absent; the data persists unless noted):

| Number | Persisted in | M / P (prod, E13) | Notes |
|---|---|---|---|
| PYQs attempted | `pyq_attempts` | M 0 in 30d / 10 all-time; P 1 / 93 | Only proxied as `/pyq/:id/submit` route counts in sessions (30d) |
| Mains submissions | `mains_evaluations` | M 0; P 1 | Since 2026-09-16 |
| Chat messages | `daily_sessions.chat_count` (client-reported, per IST day) | M 1; P 8 | Plus `auth_events.chat_conversation_started` (90d) |
| Days studied | `daily_sessions` rows | M 1; P 4 in 30d | — |
| Days opened app | `api_usage_daily` | M 6; P 20 in 30d | 120-day rollup |
| Notifications tapped | `notification_dispatch.tapped_at` | 0 / 0 | — |
| Signup source | `link_events.source` + `user_id` (claim) | — | — |
| Psychometric completed/skipped | `profile.psychometricTestCompleted/Skipped` | — | In `GET :id`, unrendered |
| UI language | `uiLanguage` | — | In `GET :id`, unrendered |
| Mandate attempts | `subscriptions` | — | Unread by any SME route |

### A.2 Exam scoping on this page

| Element | Exam behaviour (code) | Verdict |
|---|---|---|
| Entitlement cards | per row `examId` (`SH:255–289`) | correct; the only "what did they buy" answer |
| Account-wide pill / `subscriptionSource` | person-level; tooltip warns (`SH:322–343`) | correctly caveated |
| Recent orders | per-order `examId` chip (`SH:195–205`) | correct |
| Quota | `activeExamId ‖ upsc-cse` only (`AS:216–217`) | an APPSC holder whose active exam is UPSC sees UPSC caps only. Labelled with the exam chip (`SH:424–431`). OK |
| Trial | person-level clock, length from `trialExamId` (`SH:360–364`) | OK |
| Timeline / sessions | exam on orders / api_usage / feedback only (`AS` `item()` callers); focus **highlights**, never filters (`ExamFocusBar.tsx:1–5`) | OK. `user_content.exam_id` exists but is not emitted (§C) |
| Incidents | `?exam=` narrows api_usage only; NULL-exam rows excluded (`AS:1203–1219`; caveat `IP:157–166`) | honest |
| **Mandates** | `subscriptions.exam_id` exists (`schema.prisma` Subscription, "Defaulted … upsc-cse") | **not read at all** |

---

## B. MEANING

> Owner ruling 2026-11-03 (verbatim): "The numbers are there BUT those numbers are not explaining anything — same on the aspirant profile screen. No need to drastically change stuff; compact the details or combine and show. I don't need to see the endpoints a user is hitting on the portal, it explains nothing. Instead explain in simple terms: some X metric done on this data, or 'he stayed X mins', or 'he has been using this many times'. I may be grossly oversimplifying — the portal might already do some things well and corrections may not be needed at all. Just analyze and judge."

**Verdict in one line.** The profile is the best-built screen in the portal:
- honest null-vs-zero;
- IST throughout;
- per-exam holdings;
- retention caveats;
- a sessions view that already turns requests into sittings.

But it fails the support sentence in three places:
1. **Money.** It cannot say "tried to pay and failed", and it calls comps "PAID ₹599".
2. **Usage.** By default it shows a partial-read estimate that understates by 3–20×, and it never says what was *studied*.
3. **Incidents.** It speaks in routes and HTTP codes.

Every number in the quoted sentences below is the **real prod value for M or P**, from §A/§E.

### B.1 Per card: plain-language test and bucket

| # | Card (UI lines) | The one sentence a support person can say today | Bucket | Minimal change |
|---|---|---|---|---|
| 1 | **Identity**: support code, name, exam chip, email/phone, status, provider, role, onboarded, platforms, member since (`SH:77–166`) | "This is the UPSC aspirant with code 79D9-…, Google sign-in, joined 23 Apr, onboarded." Passes. | **KEEP AS IS** | Optional: relabel "Platforms" → "Push devices" (A#17). |
| 2 | **What they hold · per exam** (`SH:291–314`) | P: "He holds UPSC and APPSC, both staff-granted, until 3 Dec." Passes. | **KEEP** (fix copy) | Replace the empty-state copy for non-trial users with the trial-ended / mandate sentence (row 4). Add `accessTier` (free/paid exam) to the card text, not only the tooltip. |
| 3 | **Account-wide** pill, premium expires, trial, trial length from (`SH:316–365`) | "Account-wide he is Premium until 3 Dec; not on trial." Passes, but repeats card 2. | **COMPACT/COMBINE** into card 2's header sentence | "Premium until 3 Dec (staff comp) · trial ended 7 May" / "Trial · 4 days left (UPSC policy)". |
| 4 | **Recent orders (last 5)** (`SH:367–376`) | Today, M: "**No orders**" — **false for support**: she tried to pay 3 times. P: "He paid ₹599 + ₹499 today": **false**, they were comps. | **COMPACT/COMBINE + derive** | Rename the block **"Money"**. One line per *attempt*, newest first, merging three sources:<br>• `orders` (PAID / FAILED / CREATED, with MANUAL rendered "**staff comp (no payment)**");<br>• `subscriptions` (mandate created → charged / abandoned / cancelled);<br>• `auth_events` `checkout_opened` / `checkout_abandoned` / `purchase_failed`.<br>Sentence M: "**Tried UPSC Monthly UPI AutoPay on 24 Sep, 2 Nov and 3 Nov (Annual); never authorised; app refused 28 retries ('already active' block, fixed 3 Nov).**"<br>Sentence P: "**Paid Apple ₹599/mo Apr–Aug; staff comps since Sep; started a Razorpay mandate on 2 Nov that was never charged.**" Data persists (B.1a M11–M12). |
| 5 | **Today (IST) quota** (`SH:410–453`) | M: "She has used none of her 3 free AI chats today." Passes for `daily` rows; wrong footer for `lifetime` rows (A#8). | **KEEP** (relabel) | Footer only on daily rows. For premium: one line "Unlimited (premium/trial) — no counters", not 10 "unlimited" rows. |
| 6 | **Streak** (`SH:455–473`) | P: "2-day streak": **false**, it broke on 17 Sep. | **KEEP** (re-source + relabel) | Read Postgres `user_auth` (authoritative). When `last_streak_date < yesterday (IST)` say "**No active streak · best 2 · last credited 17 Sep**". |
| 7 | **Five last-seen clocks** (`SH:480–534`) | Needs five columns to say "last opened the app 1 h ago". Raw route and event-type strings. | **COMPACT/COMBINE** | One line: "**Last opened the app 1 h ago (iOS 2.1, 3 devices) · push on**". Source: `sessions.lastAccessedAt` (A#12) + versions + tokens. Keep the five clocks behind a "details" disclosure. Drop route / event-type text from the headline. |
| 8 | **Versions + devices + push tokens** (`SH:537–561`) | "Android 2.1, 1 device, push granted." Passes. | **COMPACT** into row 7's line | — |
| 9 | **Time in app · last 30d** (`SH:586–635`) | Right idea, wrong default. P reads "≥2 sessions, ≥14–33 min, ≥1 day" when the truth is **34 sittings, ~2 h 27 min, 21 of 30 days**. Two-method ranges ("pairs … gap") are not a sentence. | **COMPACT + compute server-side** | One sentence: "**Opened the app on 21 of the last 30 days, ~34 sittings, ~2½ h in total (≈7 min a day when active)**". Compute in SQL over the full 30 days, the same 30-min rule (M2, M3), so no partial read and no "Load full" button. Keep the range only in the tooltip. |
| 10 | **Activity · N events + 16 count tiles** (`SH:639–694`) | M: "708 events": none. "593 Requests", "0 Payment webhooks", "0 Psychometric tests" explain nothing. | **RAW/NOISE** → replace | Replace the total and the 16 tiles with **one "What they did (last 30 days)" sentence**: "**1 PYQ practised (93 all-time), 1 mains answer evaluated, 8 AI-chat messages, 1 doc question; studied on 4 days; 59 pushes received, 0 tapped**" (M4–M9). Keep the tiles as timeline *filters* (they are good doorways), minus the false "Payment webhooks" tile. |
| 11 | **Exam focus bar** (`ExamFocusBar.tsx`) | "Highlight his APPSC activity." Passes as a tool; no number. | **KEEP AS IS** | — |
| 12 | **Incidents** (`IP:25–237`) | M: "46 failures on POST /api/v1/payments/subscriptions": none. The sentence that is true is "**she tapped Buy 23 times in 10 min on 2 Nov and the server refused each one**". The empty state "Nothing broke — you can say so" is false for payment trouble. | **COMPACT/COMBINE (relabel)** | Count **attempts, not rows**: collapse a 400 and the `purchase_failed` within the same few seconds into one. Map route → feature ("Buy UPSC subscription", "AI Mentor chat"). Render 402 as "hit a limit/paywall", not a failure. Add `orders.FAILED`, `payments.FAILED` and abandoned mandates to the feed, or soften the empty-state claim to "no app/server errors recorded". |
| 13 | **Timeline · Sessions view** (`SV:196–315`) | P/M: "13:53–13:59 IST, opened the app for ~6 min, tried to buy, abandoned." The row almost says it, but leads with "258 requests", "UPSC ×227 APPSC ×31" and 54 raw event names. | **KEEP + relabel chips** | Drop the "N requests" and per-exam request-count chips from the collapsed row; keep them inside. Collapse repeated key events ("purchase_failed ×22, upgrade_tapped ×23" → "**tried to buy 23×, failed**"). Fix "sign-in/client errors" to say "purchase failed" when that is the event. Use `pyq_attempts` for PYQ counts (M5) instead of route-sniffing. |
| 14 | **Timeline · Events view + source filter** (`TL:58–152`) | An engineer's view (17 table names, "request noise — opt-in"). Fine as a drill-down. | **KEEP AS IS** (secondary) | — |
| 15 | **Staff notes** (`NotesComposer.tsx`) | "On 3 Nov staff comped UPSC+APPSC after the failed mandates." Passes, but P's 4 comps carry **empty** `actor_note`. | **KEEP AS IS** | (Upstream: the grant modal's `reference` is optional.) |

#### B.1a Derived metrics proposed above: computable today?

| Id | Metric | Today? | Table.column and rule | Cap / caveat |
|---|---|---|---|---|
| M1 | Last opened the app | **Yes** | `max(sessions."lastAccessedAt")` per user | full history; any authenticated request |
| M2 | Sittings (sessions), 30d | **Yes** | Union `api_usage.created_at` + `auth_events.created_at` per user, ordered. A new session starts when the gap to the previous row is > 30 min (`SU:26`; E7 implements it) | 30d raw cap; guard-refused requests invisible |
| M3 | Minutes in app, 30d | **Estimable** | Sum of (last − first) per M2 session; or the sum of `app_opened`→`app_backgrounded` pairs ≤ 6 h (`SU:333–362`) | Lower bound. **Exact foreground time needs new persistence** (flag only) |
| M4 | Days opened app | **Yes** | `count(DISTINCT day)` from `api_usage_daily` per user | 120 days |
| M5 | PYQs practised | **Yes** | `pyq_attempts`: `count(*)`, `count(DISTINCT question_id)`, `count(*) FILTER (WHERE is_correct)`, re-attempts `attempt_number>1` | from 2026-08-26 (backfilled from Neo4j) |
| M6 | Mains answers evaluated | **Yes** | `count(*)` from `mains_evaluations` | from 2026-09-16 (answer text not stored) |
| M7 | AI-chat use | **Partly** | `sum(daily_sessions.chat_count)` (client-reported per IST day); conversations = `auth_events` `chat_conversation_started` | daily_sessions full history (client-dependent); auth_events 90d. Message content: no persistence |
| M8 | Days studied | **Yes** | `count(DISTINCT d)` over `daily_sessions.date` ∪ IST dates of `pyq_attempts`, `user_question_attempts.updated_at`, `user_document_progress.updated_at`, `simulation_attempts."startedAt"`, `mains_evaluations.created_at`, `user_content.created_at` | full history; `updated_at` keeps only the latest touch |
| M9 | Pushes received / tapped | **Yes** | `notification_dispatch` (`sent_at`, `tapped_at`) for rule pushes; `notification_history` for feed rows | — |
| M10 | Streak (true current) | **Yes** | `user_auth.current_streak` when `last_streak_date >= (IST today − 1)`, else 0; `max_streak` | full |
| M11 | **Checkout / mandate attempts** | **Yes** | `subscriptions` per user (`provider`, `status`, `plan_type`, `exam_id`, `created_at`, `first_charge_granted_at`). Never charged = `first_charge_granted_at IS NULL`. Plus `orders` with `status IN ('CREATED','FAILED')`. Plus retries = `api_usage` `POST /api/v1/payments/subscriptions|orders` with status ≥ 400 (30d). Plus `auth_events` `checkout_opened` / `checkout_abandoned` / `purchase_failed` (90d, app 2.0+) | The *reason* for a 4xx is **not persisted** (no response body in `api_usage`; `purchase_failed.error_code/message` null in prod). Needs new persistence (flag only) |
| M12 | Comp vs paid | **Yes** | `orders.payment_source = 'MANUAL'` → "staff comp"; `sme_audit_log` `PREMIUM_GRANT` (`actor_note`) | — |

### B.2 The six shareholder questions on this screen

| Q | Answered? | Missing | Persists? |
|---|---|---|---|
| (1) Feature use + repeat rate (one user) | **Partially.** Sessions and time-in-app exist; *what was studied* does not (no PYQ / mains / chat counts). | M5–M8 | Yes (`pyq_attempts`, `mains_evaluations`, `daily_sessions`) |
| (2) Per-exam and per-language funnel | **Exam yes, language no.** | `uiLanguage` unrendered | Yes (`user_profiles.ui_language`) |
| (3) Where buyers stop in checkout | **Not answered for this user.** Mandates are invisible; only incident route counts hint at it. | M11 | Yes (`subscriptions`, `orders`, `auth_events`); reason text no |
| (4) Question quality | n/a | — | — |
| (5) Notification outcomes | **Partially.** Read/unread only. | M9 tapped | Yes |
| (6) **Support view incl. Razorpay mandates** | **Not answered for mandates; wrong for comps.** This is the owner's core complaint, now proven on M and P. | M11, M12 | Yes, all in Postgres |

### B.3 What the profile must show for exam + language, per entitlement

Per the coordinator, addendum 2 (filter bar) applies to the list screen. For this screen:

**Each holding line should read:**
"**{Exam} · {paid|free exam} · {LIVE until d / EXPIRED d / REVOKED} · {Apple | Razorpay | Staff comp} · {last money event: paid ₹x on d | mandate pending since d | comp by staff on d}**".

| Field | Source |
|---|---|
| Exam, LIVE/EXPIRED/REVOKED, source | already in `entitlements[]` |
| paid/free exam | `accessTier` (today only in the tooltip) |
| last money event | join by `exam_id` to `orders` / `subscriptions`; both carry `exam_id` |

Exams with **no holding but an attempt** must also appear, from `subscriptions` / `orders` by `exam_id`. Example: "UPSC · tried Annual AutoPay 3 Nov · never authorised".

**Person-level, once:**
- active exam, already shown;
- **UI language** (`uiLanguage`, null = "device/English, never chosen");
- WhatsApp exams (`whatsappExamsEffective`).

These are already returned by `GET /sme/users/:id` (`US:182–193`, `:760–764`) and simply not rendered. Content-language per exam is not tracked per user (flag only).

---

## C. GAP: documented fields and params not rendered

| Field (doc) | Backend | Proxy | Profile UI | Status |
|---|---|---|---|---|
| `GET :id` `recentOrders[].latestPayment` {status, method} (SME_PORTAL_API §1.2) | `US:220–228` | passthrough M:4641 | page uses the snapshot's orders | proxied-only |
| `GET :id` `recentOrders[].isTest` | `US:218` | ✔ | snapshot lacks it → chip dead (`SH:206`) | proxied-only / **broken** |
| `GET :id` `profile.*` (psychometricTestCompleted/Skipped, aspirantType, attempt_year, medium…) | `US:194` | ✔ | only `onboardingCompleted` | proxied-only |
| `uiLanguage` | `US:760–764` | ✔ | absent | proxied-only |
| `whatsappExamIds` / `whatsappExamsEffective` | `US:183–193` | ✔ | absent on the profile | proxied-only |
| `trialExtended` | `US:192` | ✔ | drawer only (`AppUsers.tsx:1059`) | proxied-only on this page |
| `entitlements[].accessTier` | ✔ | ✔ | tooltip only (`SH:260`) | rendered (tooltip) |
| `entitlements[].isTrial` / `trialEndsAt` | ✔ | ✔ | `SH:282–286` | **rendered** (contradicts C2 for this page) |
| `appleOriginalTransactionId` "visible in detail" (SME_APPLE_IAP_API §2/§4) | never emitted (C2) | — | — | absent (not re-verified) |
| `subscriptions` (mandates) | **no SME route** | — | — | absent: needs a backend read (column data exists) |
| `pyq_attempts`, `mains_evaluations`, `daily_sessions`, `notification_dispatch.tapped_at` as timeline / count sources | not in `ACTIVITY_SOURCES` (dto `:20–38`) | — | — | absent |
| `user_content.exam_id` on timeline items | column exists; `fetchUserContent` passes no `exam` (`AS:1012–1027`) | — | ExamFocusBar copy "only orders, requests and feedback record an exam" (`ExamFocusBar.tsx:53–57`) | absent / stale copy |
| P7 "older build" empty-state copy (C1) | — | — | generic | partial (C1) |

**Stale or false UI copy:**
- "there is no PYQ-attempt source upstream" (`SV:129`, `ActivityItem.tsx:150`, `SU:4–8`): the table exists; the timeline does not expose it.
- "resets at midnight IST — no history is kept" on lifetime rows (`SH:444–446`).
- "Nothing broke for this user — you can say so" (`IP:207`).
- "Payment webhooks" label (`DU:172`): structurally the client-verify ledger only.
- "isPremium: true — entitlement active" (`SH:331`).
- "normal for a trial user" for trial-ended users (`SH:305`).

---

## D. Contradictions (code vs docs / brief / earlier reports)

1. **SME_ACTIVITY_TRAIL_API.md says webhooks are visible per user.**
   - Doc L566: `payment_events` is "the per-user view of the same provider traffic". L532 example type `razorpay.webhook`. L598: "Did the payment webhook land before or after they said they paid?".
   - Code: webhooks are recorded **without `userId`/`orderId`** (`PS:819–826`; `apple-payments.service.ts:848–854`), and `markPaymentEventProcessed` never backfills (`PS:276–290`).
   - Prod: 0 / 33,370 Razorpay and 0 / 60 Apple webhook rows are attributable. **The question in L598 cannot be answered on this page.**
2. **Streak source.**
   - SME_ACTIVITY_TRAIL_API §limitation 4 (L177) and the snapshot (`AS:544–574`) read Neo4j.
   - `schema.prisma:91–99` and `streak.service.ts:210–218`: "Postgres remains authoritative and nothing reads the graph copy back — do not reintroduce a read against it." The snapshot is that read.
3. **Brief / SHARED_RULES planes.** "Chat / mains-eval / PYQ-variation have ZERO persistence". Partly false:
   - `mains_evaluations` persists every evaluation (`schema.prisma` MainsEvaluation; writer `mains.service.ts:541`; since 2026-09-16; P has 1 row).
   - `daily_sessions.chat_count` persists per-day chat counts (client-reported).
   - True only for chat *content* and PYQ variations.
4. **Coordinator note.** "ab74a3b2 = paid Razorpay subscriber" is false. He is an ex-Apple subscriber with 4 MANUAL comps and an uncharged PENDING Razorpay mandate (E3, E2).
5. **AUDIT_2026-11 §1.2** `/app-users/:id` row omits `GET /sme/users/{id}`, which the page calls (`AUD:134–139`; live capture line 177). It also omits the notes read `timeline?sources=notes` (capture line 175).
6. **C2 row 8 / "isTrial never shown".** True for `AppUsers.tsx` only. The profile renders `isTrial` (`SH:282–286`) and shows `accessTier` in a tooltip (`SH:260`).
7. **C2 row 12** says `trialExtended` is rendered at `AppUserDetail.tsx:136`. That line only fetches; the profile never renders it.
8. **C2 row 14** says the detail `recentOrders` `isTest` is "RENDERED (equivalent elsewhere)" via the snapshot. The snapshot never emits `isTest` (`AS:288–297`; dto `:427–438`), so the chip is dead.
9. **Snapshot ApiOperation** "TODAY-only quota (Redis, midnight-IST TTL)" (`sme-activity.controller.ts:50–51`) and the DTO comment. `pyq_open` is a Postgres lifetime counter (`quota.service.ts:114–130`).
10. **sme-activity.dto.ts:11–12** "Errors are the Nest default `{statusCode, message}`". Errors are actually the 7-key `{success:false,…}` envelope (SME_PORTAL_API §1.7, captured). Same class as C1 contradiction #7.
11. **"Incidents — derived only, failures"** (`sme-activity.controller.ts:79–82`). The panel UI claims "Nothing broke" (`IP:207`), but money failures (`orders/payments FAILED`, mandate abandonment) are outside the definition.

---

## E. Prod SQL (read-only; RUN 2026-11-03 after the 14:13 IST deploy; results inline)

`:uid` = the user id. Internal guard (the coordinator's set) is applied wherever a user row is read:
`NOT (u.email ILIKE '%@stanzasoft.%' OR u.email ILIKE '%@prepmonkey.%' OR u.email ILIKE '%@designasylum.in')`. Both M and P returned `internal = f`.

| # | Query (abridged; exact text was run) | Result M (3a5a9b8f) | Result P (ab74a3b2) |
|---|---|---|---|
| E1 | `SELECT status, internal-flag, "createdAt", "lastLoginAt", last_active_at, trial_ends_at, premium_expires_at FROM user_auth WHERE id=:uid` | UNSUBSCRIBED; created 04-23; login 08-25; active 11-03 05:49; no premium | SUBSCRIBED; premium to 12-03 05:26; source MANUAL |
| E2 | `SELECT provider,status,plan_type,exam_id,offer_code,first_charge_paise,first_charge_granted_at,created_at FROM subscriptions WHERE user_id=:uid` | 3 rows: 09-24 MONTHLY CANCELLED; 11-02 MONTHLY CANCELLED; 11-03 ANNUAL PENDING; none charged; offer_code null | 1 row: 11-02 MONTHLY PENDING, never charged |
| E3 | `SELECT status,amount,payment_source,exam_id,is_test,apple_environment,created_at,premium_granted_at,(payments) FROM orders WHERE "userId"=:uid` | 0 rows | 8: 4 APPLE Production ₹599 (Apr–Aug, `apple_iap` CAPTURED) + 4 MANUAL (09-16, 09-17, 11-03 UPSC ₹599, 11-03 APPSC ₹499; payment `manual`) |
| E4 | `SELECT exam_id,source,expires_at,revoked_at FROM user_exam_entitlements WHERE user_id=:uid` | 0 rows | upsc-cse MANUAL → 12-03; appsc-group-1 MANUAL → 12-03 |
| E5 | `SELECT current_streak,max_streak,last_streak_date FROM user_auth WHERE id=:uid` | 0 / 0 / null (= UI) | 2 / 2 / 2026-09-17 (= UI) |
| E6 | `SELECT max("lastAccessedAt") FROM sessions WHERE "userId"=:uid` | 11-03 05:49:53 UTC (= "3h ago") | 11-03 07:43 UTC (= "1h ago") |
| E7 | 30-min-gap sessionisation over `api_usage ∪ auth_events`, last 30 days (window `lag()`, new session when gap > 30 min or first row; minutes = Σ(max − min); days = distinct IST dates) | **8 sessions / 15 min / 7 days** (UI ≥2–3 / ≥1–12 / ≥1–3) | **34 / 147 min / 21 days** (UI ≥2 / ≥14–33 / ≥1) |
| E8 | per-source 30-day counts mirroring `AS:427–503` (e.g. `SELECT count(*) FROM notification_history WHERE "userId"=:uid AND "createdAt">=now()-interval '30 days'`) | notifications 32 (= UI) | 59 (= UI) |
| E9 | `SELECT provider,kind,count(*),count(user_id),count(order_id) FROM payment_events GROUP BY 1,2` | apple client_verify 9/9/0; apple webhook 60/**0**/**0**; razorpay client_verify 5/5/0; razorpay webhook 33,370/**0**/**0** | same (global) |
| E10 | `SELECT IST time, status, exam_id, app_version FROM api_usage WHERE user_id IN (:M,:P) AND method='POST' AND route='/api/v1/payments/subscriptions' ORDER BY created_at` | 11-02: 1×200 at 13:53:32 then 24×400 13:54:07–14:03:12; 11-03: 200 at 11:19:44 then 4×400 11:19:49–54 | 11-02: 200 at 14:04:52 then 2×400 at 14:05:27 |
| E11 | `SELECT IST day, event_type, error_code, left(message,140), count(*) FROM auth_events WHERE user_id=:uid AND event_type IN ('purchase_failed','checkout_opened','checkout_abandoned','upgrade_tapped','paywall_viewed') GROUP BY 1,2,3,4` | 09-24 opened/abandoned ×1; 11-02 purchase_failed 22, upgrade_tapped 23, opened/abandoned 1; 11-03 purchase_failed 4; **error_code/message all null** | (not run; incident shows purchase_failed ×2) |
| E12 | `SELECT count(*) FROM orders WHERE "userId"=:uid AND status='FAILED'` plus `payments … status='FAILED'` (feeds A#24) | 0 | 0 |
| E13 | pyq_attempts (30d / all), mains_evaluations, daily_sessions days + Σchat_count (30d IST), api_usage_daily distinct days (30d), notification_dispatch sent / tapped (30d) | 0 / 10; 0; 1 day, 1 chat; 6 days; 32 / 0 | 1 / 93; 1; 4 days, 8 chats; 20 days; 59 / 0 |
| E14 | `SELECT count(DISTINCT s.user_id), count(*), count(DISTINCT s.user_id) FILTER (WHERE NOT EXISTS orders) FROM subscriptions s JOIN user_auth u … WHERE provider='RAZORPAY' AND first_charge_granted_at IS NULL AND <internal guard>` | **47 users / 61 mandates / 42 with zero orders** (46 PENDING, 15 CANCELLED) | — |
| E15 | `SELECT count(*), count(DISTINCT "userId"), sum(amount)/100 FROM orders WHERE payment_source='MANUAL' AND status='PAID'` | 36 / 13 / ₹46,168 notional | — |
| E16 | `SELECT count(*), count(DISTINCT "userId"), non-internal users FROM orders o JOIN user_auth u … WHERE o.is_test` | 4 / 1 / 0 non-internal | — |
| E0 | `SELECT min(created_at) FROM api_usage` (retention check) | 2026-10-03 20:01 UTC → purge running at 30d | — |

**Processes:**
- No long-running process was started.
- Each `npx ts-node -T scripts/staging-db.ts --prod` call (~14) ran in the foreground and exited.
- No pids remain.
- No browser was used; the live captures were made by the coordinator.

**Files written:** this report only, `/private/tmp/claude-502/-Users-yashwanth-Desktop-projects/b0942abe-5468-460e-92b6-db352e6a3606/scratchpad/phase1/S03_app_user_detail.md`.
