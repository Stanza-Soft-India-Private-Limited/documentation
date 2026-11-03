# S04 — `/app-users` aspirant LIST (Phase 1 verifier)

**Paths**
- `UI:` = `portal/admin-ui/src/`. `AU` = `pages/AppUsers.tsx`. `API` = `services/api.ts`.
- `M:` = `portal/app/main.py`.
- `US` = `backend/src/modules/sme/services/sme-user.service.ts`. `DTO` = `backend/src/modules/sme/dto/sme-user.dto.ts`. `PCU` = `backend/src/common/utils/premium-check.util.ts`.

**Versions and evidence**
- Backend: local HEAD `cca9ede`. Prod image deployed 14:13 IST today.
- Portal: clone `7b00daf` (2026-10-27). Live is v2.44.0. The live aria tree matches the clone line for line (Status options, labels, columns).
- Live capture: `.playwright-mcp/live/app_users_{network.txt,snapshot.yml,full.png}`.
  - On open the page made two calls: `GET /sme/users?page=1&limit=20` and `GET /sme/exams` (network L156–157).
- Prod SQL: read-only, through `scripts/staging-db.ts --prod`. The SQL files are in the scratchpad (`s04_q1…q6.sql`).
- Counts marked **ext** exclude internal accounts (`@stanzasoft.%`, `@prepmonkey.%`, `@designasylum.in`; 17 accounts).
- No names, emails or phones appear in this report.

---

## Headline (read first)

1. **🔴 The Platform filter returns nothing whenever it is combined with Premium, Premium state or a Joined date. Export CSV with any Platform set always downloads an empty file.**
   - Cause: those filters move the request to the full-scan path. There, Platform is matched against `provider`/`subscriptionSource` (M:4497–4500).
   - Production has no row where either field equals `ios`, `android` or `web`:
     - `provider` is only `google` (14,559) or `apple` (1,685);
     - `subscription_source` is only `APPLE`, `RAZORPAY`, `MANUAL` or null.
   - On the fast path, "iOS" means an active iOS push token (`US:119–123`). That is 4,893 ext users, and **151** of them are premium by the portal's own definition.
   - Export always takes the scan path (M:4610), so "Export CSV" of the list view "Platform = iOS" produces a header-only file.

2. **🟠 Three premium-ish filters, three meanings, and none of them says "paid".**

   | Control | Real meaning | ext count today |
   |---|---|---|
   | **Status=SUBSCRIBED** | stale `user_auth.status` mirror | 20 |
   | **Premium=yes** | `isPremium` = paid mirror **or** in trial | **294**, of which 274 are trialists |
   | **Premium state** | funnel label | 20 Premium / 274 Trial / 15,751 Trial Ended / 12 Churned / 170 Downloaded |

   The truth is that **20** ext users hold a LIVE entitlement. Only **16** paid money; 4 are staff comps.

3. **🟠 Status is a stale mirror, not a state.**
   - **7,993** ext users show `ACTIVE` but their trial ended long ago. `PremiumExpiryGuard` only flips the column when the user makes a request.
   - 5,020 users are `ONBOARDING`, and the Status filter cannot select them: the option is missing (`AU:84–86`), and our DTO enum rejects the value (`DTO:17–24`).
   - `INACTIVE` and `LOCKED` match 0 users.

4. **🟠 "Send notification → Cohort: Trial" reaches 1,609 users. The list calls only 274 of them "Trial".**
   - `resolveSegment` uses a hard-coded 14-day window (`US:645, 663–673`).
   - UPSC trials are 3 days, APPSC trials are 0.
   - The UI hint "length is set per exam" (`AU:624`) is false. So is the doc (`SME_NOTIFICATIONS_API.md` L218).
   - Of the 1,609, **1,335** are users the list labels "Trial Ended".

5. **Exam works.** It is the one correct axis:
   - Exam mode uses NULL → UPSC.
   - "Holds X" (Premium with an exam selected) equals the entitlement truth: UPSC 15, APPSC 7 ext.
   - **But** UPSC = 15,031 people, of whom only **3,383** ever chose UPSC.

---

## A. TRUST

### A.0 The two code paths (verified in code)

| Path | Triggered when | What runs | Count / slice | Platform means | Premium means |
|---|---|---|---|---|---|
| **FAST** (passthrough) | none of `premium`, `premiumState`, `createdFrom`, `createdTo` is set (M:4560) | `_sme_proxy GET /sme/users` with page, limit, search, status, onboarded, platform, exam (M:4574–4586). Ours: Prisma `where` + `count` (`US:64–157`), `orderBy createdAt desc` | our `total` (`US:142`) | active device token (`US:119–123`) | n/a (never forwarded) |
| **SCAN** (full scan + local filter) | **any** of `premium` / `premiumState` / `createdFrom` / `createdTo` (M:4560) — this includes every Premium, Premium state and Joined selection | `_sme_fetch_all('/sme/users', params=_sme_users_corpus_params(exam, premium))` (M:4561). The corpus is all users, or `{exam}` narrowed **unless** `exam` + `premium=true` (M:4440–4450). Then `_sme_user_matches` (M:4466–4522), then a local slice (M:4563–4572) | `len(filtered)`, plus `coverage` | `provider` / `subscriptionSource` string equality (M:4497–4500) → **always 0 rows** | no exam: `isPremium` incl. trial (M:4489). With exam: LIVE entitlement for that exam (M:4485–4488, `_sme_holds_live` M:4427–4437) |
| **EXPORT** | always (M:4589–4630) | always a SCAN, whatever the filters | n/a | as SCAN | as SCAN |

**Scan-path mechanics (`_sme_fetch_all`, M:5524–5579):**
- **Size.** 100 rows per page; 16,244 rows means **163 upstream calls**, in batches of 20.
- **Page changes are not cached.** `queryKey: ['sme-users', filters]` includes `page` (`AU:1230–1233`), so every click re-scans the whole corpus: 163 calls per click.
- **Failed pages.**
  - The first page raises.
  - A later failed page is counted and **dropped**.
  - `complete = not truncated and failedPages == 0` (M:5577). It never compares `fetched` with `total`.
  - The list shows an amber "floor" banner when `!complete` (`AU:1528–1537`).
  - **Export discards `meta` (`rows, _ =`, M:4610), so a partial CSV is silent.** 🟠
- **Offset race.** Pagination is OFFSET with `createdAt desc`. A signup during the scan (104 ext today) shifts later pages: one row is duplicated and the oldest is missed, and `complete` stays true. 🟡 (rare: the scan takes seconds)
- **Cap.** 400 pages × 100 = 40,000 rows. Not reached today.
- **Bug class 1** (query string on path): not applicable. `_sme_proxy` merges (M:4356–4366) and no path carries a query.
- **Bug class 7** (repeated keys): not applicable. FastAPI declares explicit `Query()` params (M:4525–4538).

### A.1 Per-filter table

| # | Filter (UI file:line) | Our definition | Proxy transformation | Window / TZ | Exclusions | Prod SQL (ext unless noted) | Verdict | Sev |
|---|---|---|---|---|---|---|---|---|
| F1 | **Search** "email / name / phone" (`AU:1386–1397`) | `OR` of ILIKE on email, username, name, phoneNumber (`US:77–84`) | FAST: forwarded. SCAN: local substring over the same 4 fields joined by spaces (M:4517–4521). The two are equivalent except for strings that span fields | — | none | — | **TRUE.** No support-code search here (it lives in the header box) | ⚪ |
| F2 | **Status** ACTIVE / INACTIVE / SUSPENDED / LOCKED / SUBSCRIBED / UNSUBSCRIBED (`AU:84–86`, `1398–1408`; live snapshot L140–145) | `where.status = enum` (`US:85`). DTO enum has **no ONBOARDING** (`DTO:17–24`) | FAST: forwarded. SCAN: local upper-case compare (M:4493–4494) | none: a raw column | none | ACTIVE 8,267 (**274** in trial, **7,993** trial ended); UNSUBSCRIBED 2,918; SUBSCRIBED 20; SUSPENDED 2; INACTIVE 0; LOCKED 0; **ONBOARDING 5,020, not selectable** (E1, E2) | **RAW / MISLABELLED.** A lifecycle mirror that `PremiumExpiryGuard` refreshes only on an authenticated request (`PCU:277–279` comment). "ACTIVE" ≠ active, ≠ in trial. Two options match nobody. The largest real bucket is unreachable | 🟠 |
| F3 | **Premium** All / "Premium (any exam, incl. trial)" / "Non-premium". With an exam selected the label becomes "Holds {exam}", with options "Holds LIVE X" / "In X, holds none" (`AU:1409–1436`) | Doc and ours: `premium=true` without exam = SUBSCRIBED and not expired (`US:107–114`); with exam = LIVE entitlement in that exam (`US:93–106`). **`premium=false` is a no-op** on our side (only `=== true` is handled, `US:92`) | **Always SCAN; never forwarded** (M:4446–4450). No exam: `isPremium` (paid mirror **or** in-window trial, `US:728, 749`; `PCU:117–129`). With exam: `_sme_holds_live`. false + exam: in exam-mode X and holds none of X (M:4513–4516) | person-level; trial uses activeExamId's length | none (internal included) | No exam: SCAN yes = **294** (paid mirror 20 + trial 274). Upstream would have said 20. **Truth: LIVE entitlement = 20; non-MANUAL = 16.** No = 15,933. With exam: UPSC holders 15, APPSC holders 7 (1 of them in UPSC mode) = truth (E3) | No exam: **MISLABELLED**. The tooltip and option are honest ("incl. trial"), but a "Premium" filter where **93 %** of hits are trialists answers no shareholder question. With exam: **TRUE**, the only correct "who holds X" in the portal | 🟠 |
| F4 | **Premium state** Premium / Trial / Trial ended / Churned / Downloaded (`AU:1437–1451`) | `derivePremiumState` (`PCU:302–325`), with the trial length of `activeExamId ‖ upsc-cse` (`US:720–723, 752`). Ours **400s** on this param (portal comment M:4473) | **Always SCAN**; exact string match (M:4492) | person-level; "Trial" uses the exam-mode's length | none | Premium 20 · Trial 274 · **Trial Ended 15,751** · Churned 12 · Downloaded 170 (E2). "Trial Ended" contains **4,850 who never finished onboarding** (status ONBOARDING; never had access, since `hasExamAccess` excludes ONBOARDING, `PCU:226–231`), plus **7,993** ACTIVE and 2,907 UNSUBSCRIBED. **81** APPSC-mode signups from the last 3 days read "Trial Ended" on day 0 (APPSC trial = 0 days) while their UPSC trial window is still open; 2 of them are on the live page 1 (E4). "Churned" = ever had `premium_expires_at`, so 3 of the 12 are lapsed staff comps (E5) | **TRUE to our derivation, MISLABELLED as a funnel.** "Trial Ended" means "not paying and not in a trial", which includes never-onboarded users and people whose exam has no trial | 🟠 |
| F5 | **Onboarded** Yes / No (`AU:1452–1463`) | `userProfile.onboardingCompleted = bool` (`US:116–118`). A profile-less user would not match `false` | FAST: forwarded. SCAN: `bool(None)` → False (M:4495), so profile-less users match "No". The paths would differ for users with no profile row | — | none | Yes 11,207 · No 5,020. Profile-less users: **0** in prod, so the paths agree today. `status=ONBOARDING` = 5,020 = Onboarded No, with 0 crossover (E3) | **TRUE.** Fully redundant with the missing Status=ONBOARDING | ⚪ |
| F6 | **Platform** iOS / Android / Web (`AU:1464–1476`) | "has an active device token" (`US:119–123`; doc §1.1 L47) | FAST: forwarded (M:4582). **SCAN: matches `provider`/`subscriptionSource` (M:4497–4500), values that never equal ios/android/web** | — | none | FAST: iOS 4,893 · Android 5,786 · **Web 0** · both 39 · no token 5,587. SCAN: **0 always**. Correct answer for iOS + Premium=yes = **151**; for iOS + joined last 7 IST days = **419** (E3, E6) | **WRONG** on SCAN and in every Export. **TRUE** on FAST, but the label "Platform" hides that it means "can receive a push on" (5,587 people with no token have no platform at all). "Web" is a dead option | 🔴 |
| F7 | **Exam** All exams / UPSC / APPSC (`AU:1477–1485`; options from `GET /sme/exams`) | `active_exam_id = X`, with **NULL and no-profile ≡ upsc-cse** (`US:86–91` → `sme-exam-scope.util.ts:26–37`); 400 on an unknown slug | FAST: forwarded. SCAN: the same rule locally, `activeExamId or "upsc-cse"` (M:4513–4516). With `premium=true` it switches to the hold axis and spans modes (M:4511) | person | none | UPSC **15,031**, of which only **3,383** explicitly chose UPSC; APPSC 1,196 (E3) | **TRUE** and documented. "UPSC" mostly means "never chose an exam" (the app predates the picker) | 🟡 |
| F8–9 | **Joined from / to** (`AU:1486–1516`) | none: our API has **no date param** (400 on any unknown key, M:4547–4553) | **Always SCAN.** UI sends `YYYY-MM-DDT00:00:00+05:30` / `T23:59:59+05:30` (`AU:81–82`, `1173–1174`); compared as instants (M:4504–4512) | **IST correct** | none | — (code-verified; 104 ext signed up today IST) | **TRUE.** Costs a full scan and silently breaks Platform (F6) | 🟡 |
| F10 | **Export CSV** (`AU:1304–1317`, `1517–1523`; `API:2095–2109`) | — (portal construct) | **Always SCAN** (M:4610), **meta discarded**. Columns: id, name, email, username, phoneNumber, provider, status, isPremium, premiumState, premiumExpiresAt, subscriptionSource, onboardingCompleted, lastLoginAt, createdAt, activeExamId, trialExamId, entitlements (M:4616–4618) | as filters | none | With any Platform set: **0 rows** (F6) | **WRONG** with any Platform (empty file). A silent partial is possible on a failed page. The `isPremium` column header carries the trial-inclusive meaning | 🔴 / 🟠 |
| H | **Header badge "16243 users"** (`AU:1357`; snapshot L127). Pager "Showing 1–20 of 16,243" (L189) | `count(*)` with the page's `where` (`US:142`) | FAST passthrough. On SCAN it becomes the filtered count | all time | **internal included; cross-exam** | All rows **16,244** now (16,243 at capture, +1 signup); **17 internal → 16,227 ext**; 5,020 never finished onboarding (E1) | **TRUE as "accounts"**, MISLABELLED as "users". The badge is not locale-formatted; the top pager is (`AU:1545`) | 🟡 |

### A.1b Per-column table (live page 1 = the 20 newest signups, all 03 Nov)

| # | Column (UI line) | Field | Live page 1 | Verdict | Sev |
|---|---|---|---|---|---|
| C1 | **User**: name/username + email + exam-mode chip (`AU:1621–1629`) | `name ‖ username`, `email`, chip only when `activeExamId` is non-null | 13 UPSC chips, 2 APPSC, 5 none (the 5 ONBOARDING) | **TRUE.** No chip = "never chose" (served UPSC): 11,648 ext users | ⚪ |
| C2 | **Premium**: premiumState badge + "Nd trial left" + LIVE entitlement chips with tooltip source/expiry (`AU:1630–1644`) | `premiumState`, `trialEndsAt`/`trialDaysLeft` (only while in trial, `US:758–761`), `entitlements[]` LIVE | 13 × "Trial · 3d trial left" (UPSC), 5 × "Downloaded" (ONBOARDING), **2 × "Trial Ended" for APPSC-mode users who joined today** | Badge **TRUE** to the derivation. Entitlement chips **TRUE** (= SQL holders). "Trial Ended" on signup day is **MISLABELLED** (APPSC has no trial; the person still has the 3-day UPSC window). MANUAL comps are indistinguishable from paid (source only in the tooltip, `AU:1639`) | 🟡 |
| C3 | **Status** (`AU:1645–1647`) | raw `user_auth.status` | 15 ACTIVE + 5 ONBOARDING | **RAW.** Same stale mirror as F2; repeats C2 in enum form | 🟡 |
| C4 | **Provider** (`AU:1648`) | `user_auth.provider` = sign-in method (google / apple). `login_provider` is null for all 16,244 | google ×18, apple ×2 | **MISLABELLED.** Sits beside a "Platform" filter but is not a platform: 3,419 Google sign-ins hold iOS tokens and 6 Apple sign-ins are on Android (E6) | 🟡 |
| C5 | **Joined** (`AU:1649`) | `createdAt` via `fmtDate`, which uses `toLocaleDateString('en-IN')` **without `timeZone`** (`AU:114–121`): the browser's zone, IST only by accident | "03 Nov 2026" | **TRUE** for IST browsers | ⚪ |
| C6 | **Actions**: 7–8 icons (`AU:1652–1688`) | see mutations | — | — | — |

### A.1c Drawer and row mutations (read-only review; none called)

| Action (UI) | Proxy guard | Our guard | UI confirmation | Finding |
|---|---|---|---|---|
| Quick view drawer (`AU:973–1131`) | `GET /sme/users/:id` M:4635 | 404 | — | Shows premiumState, exam mode, trial, `trialExtended`, every entitlement row. Copy "None. A trial creates no entitlement row…" (`AU:1073`) is shown also to mandate holders who tried to pay (S03 finding) |
| **Edit** name/status (`AU:170–216`) | whitelist name, status (M:4652) | `IsEnum(SmeUserStatus)` (`DTO:113–116`), audit `USER_UPDATE` (`US:260–263`) | **none**. Free status dropdown | 🟡 **(a)** Editing just the *name* of any ONBOARDING user (5,020) sends `status:'ONBOARDING'` from state (`AU:172, 175`), and our enum rejects it → 400. **(b)** Hand-setting `SUBSCRIBED` writes a paid mirror with no entitlement. The user then reads "Premium" (perpetual, `premiumExpiresAt` null), enters the `premium` push cohort and the engine's `premium` audience (`notification-audience.util.ts:139`), and is granted no access (`PCU:217–231`). Code-verified |
| Notify (`AU:569+`) | `_validate_notify_params` (M:4716–4773) | user exists; audit `USER_NOTIFY` (`US:525–543`) | modal (+ schedule) | OK (C2). Direct sends bypass prefs/quiet hours; not said on the UI |
| **Grant premium** (`AU:225–368`) | **examId required** (M:4685–4686) | `assertExists` (`US:322–323`), MANUAL order + audit (`US:324–340`) | modal; exam must be one not held LIVE (`AU:248–250`) | Amount defaults to list price → MANUAL "PAID ₹599" rows (S03 A#2) |
| **Revoke** (`AU:1735–1810`) | `exam` Query required (M:4688–4699) | `?exam` required / `all` (`US:362–373`) | per-exam radio + **second confirm** for `all` | OK |
| **Deactivate / Reactivate** (`AU:1254–1270`, `1331`, `1676`, `1110`) | — (M:4655) | SUSPENDED + kill sessions, audited (`US:270–293`) | **one click, no confirm** | 🟡 Reactivate = `PATCH status: ACTIVE` (`AU:1257`) whatever the prior status. A deactivated **payer** comes back `ACTIVE`, so the list shows Trial Ended / Churned while the entitlement still grants access |
| **Extend trial** (`AU:383–510`) | whitelist days, reason (M:6466–6474) | 400 if paid mirror, or status SUSPENDED / LOCKED / **ONBOARDING** / INACTIVE (`US:433–442`) | modal; blocked list = SUSPENDED / LOCKED / INACTIVE only (`AU:112`) | 🟡 The form is offered to the 5,020 ONBOARDING users and our side 400s (toast shows the message). The UI guard is "holds any LIVE entitlement" vs our mirror check (comment `AU:398–400` acknowledges it) |
| **Delete** (admin) (`AU:915–958`) | `require_admin` (M:4664) | audit before cascade (`US:295–308`) | type-the-email | OK |
| **Send notification** (all / cohort / selected) (`AU:628–839`) | broadcast M:4788, segment M:4805 (120 s), multi fan-out M:4827–4866 | segment `resolveSegment` (`US:622–698`) | `ConfirmModal` (`AU:817–839`) | 🟠 **Cohort "Trial" = ACTIVE ∧ (trial_ends_at future ∨ (trial_ends_at NULL ∧ never paid ∧ created ≤ 14 d))** (`US:645, 663–673`). `trial_ends_at` is NULL for every fresh signup (`PCU:19–20`; only 4 ext users ever had it set), so the window is a fixed 14 days for everyone: **1,609 recipients vs 274 "Trial" on the list; 1,335 of them the list calls "Trial Ended"** (E5). The UI hint "length is set per exam" (`AU:624`) is false. "Free" correspondingly excludes those 1,335 |
| **Scheduled** (`AU:846+`) | their DynamoDB queue (M:5008–5068) | — | — | **Out of scope (AI/their side).** Dispatch reaches our broadcast/segment/notify, so the "Trial" cohort bug applies to scheduled sends too |

### A.2 Exam scoping: which filters the Exam selector scopes

| Filter / element | Exam-scoped on OUR side? | What the portal does | Verdict |
|---|---|---|---|
| Rows / `total` | yes: `exam=` = active exam mode, NULL ≡ UPSC (`US:86–91`) | forwarded (FAST) or the same rule locally (SCAN, M:4513–4516) | correct |
| Premium (bool) | yes, **only for `true`**: with exam it becomes "holds LIVE X" (`US:93–106`) | never forwarded; re-implemented by hold check (M:4485–4488), spanning modes (M:4511). `false` + exam = "in X, holds none" (portal-only meaning) | correct result; our param unused |
| Premium state | **no** (we 400 on the param). The label is person-level, computed with the **activeExamId's** trial length (`US:720–723`) | local, account-wide | Within exam X mode it is effectively X's trial. With Exam=UPSC it includes 11,648 never-chose users on the legacy 14-day policy |
| Status | no (person-level mirror) | account-wide | — |
| Onboarded | no | account-wide | — |
| Platform | no (device tokens are not per exam) | account-wide (and broken on SCAN) | — |
| Joined | no | account-wide | — |
| Row "Premium" column | entitlement chips are per exam; `premiumState` is not | renders both side by side | A UPSC-mode user holding only APPSC shows "Premium" + an APPSC chip: correct, if read carefully |
| Header count | cross-exam unless `exam=` (doc §1.1 L206–208) | — | honest |

---

## B. MEANING

> **Owner ruling 1 (2026-11-03, verbatim):** "The numbers are there BUT those numbers are not explaining anything — same on the aspirant profile screen. No need to drastically change stuff; compact the details or combine and show. I don't need to see the endpoints a user is hitting on the portal, it explains nothing. Instead explain in simple terms: some X metric done on this data, or 'he stayed X mins', or 'he has been using this many times'. I may be grossly oversimplifying — the portal might already do some things well and corrections may not be needed at all. Just analyze and judge."
>
> **Owner ruling 2 (filter bar, verbatim):** "make this actual sense, NOT this mess — we are going to have the multi-exam thing and the language thing down the line, keep that in mind."

**Judgement.** The *data* behind this screen is mostly right and the portal already does the hard part well:
- per-exam holdings;
- IST date bounds;
- an honest coverage banner;
- an honest "incl. trial" tooltip.

The **mess is the filter model**:
- Four controls (Status, Premium, Premium state, Onboarded) describe one question, "what access does this person have?", in four vocabularies. They disagree (20 / 294 / 274+20 / 5,020).
- Two of them silently change *what another filter means* (Platform on the scan path).
- The largest real bucket (never finished sign-up, 5,020) can be reached only through a filter named "Onboarded".

### B.1 Proposed filter MODEL: four dimensions + search

**Dimension 1 · Exam (first-class; scopes everything to its right).** Values come from `GET /sme/exams` (UPSC, APPSC, …), plus "All exams". Backend `exam=` already exists (`US:86–91`).
- Selecting an exam re-scopes the Access axis to that exam. The portal already does this for Premium ("Holds X", `AU:1416`).
- Copy fix: "UPSC (incl. never chose)", since 11,648 of UPSC's 15,031 never picked an exam.

**Dimension 2 · Access (ONE axis; replaces Status + Premium + Premium state + Onboarded).**

| Value | Plain meaning | With an exam X selected → backend mapping | With "All exams" → backend mapping | Backend support today |
|---|---|---|---|---|
| **Not onboarded** | signed up, never finished onboarding: no trial, no access | `onboarded=false` (+ `exam=`) | `onboarded=false` (≡ status ONBOARDING, 5,020 = 5,020) | ✅ `onboarded` (FAST) |
| **On trial** | inside the free-trial window | `exam=X` + `premiumState=Trial`. For users *in X mode* the label already uses X's length (`US:720–723`). Exact per-exam trial for non-X-mode users needs a param | `premiumState=Trial` (274) | ⚠️ we 400 on `premiumState` → portal SCAN today. **Flag:** backend `access=trial` param |
| **Trial over, never bought** | had or could have had a trial, never held an entitlement | `exam=X` + `premiumState='Trial Ended'` minus ONBOARDING. For a 0-day exam say "**No trial (X has none)**", not "Trial Ended" | `premiumState='Trial Ended'` ∧ onboarded (10,901 ext) | ⚠️ SCAN-only. **Flag:** `access=trial_over` |
| **Paid** | holds a LIVE entitlement bought with money (APPLE / RAZORPAY) | `exam=X&premium=true` (**exists, `US:93–106`, never forwarded**) minus `source=MANUAL` | LIVE non-MANUAL in any exam (16 ext) | ⚠️ `premium=true` exists but includes comps. **Flag:** a source split (`access=paid` vs `comped`) |
| **Comped** | holds a LIVE entitlement granted by staff | `entitlements[]` LIVE ∧ `source=MANUAL` (already on every row) | 4 ext | ⚠️ derivable locally from the row; **flag** a param to avoid the scan |
| **Lapsed** (was "Churned") | held X before, nothing LIVE now | `entitlements[]` has an EXPIRED / REVOKED row for X and no LIVE row (on every row) | `premiumState=Churned` (12), but that counts lapsed comps too | ⚠️ **Flag:** `access=lapsed` (per exam needs the entitlement rows) |
| *(modifier)* **Deactivated** | staff-suspended | `status=SUSPENDED` | same (2) | ✅ `status` |

*Dropped:* Status=ACTIVE / UNSUBSCRIBED / SUBSCRIBED / INACTIVE / LOCKED (a stale mirror and two empty values); Premium yes/no (the trial-inclusive `isPremium` is not a business question); the "Premium state" control as a separate select (its values move into Access). `isPremium` stays in the API for feature gating, not as a filter.

**Dimension 3 · Language (later; first-class beside Exam).**
- Field: `user_profiles.ui_language`. It is already on every list row as `uiLanguage` (`US:771`; SME_LOCALIZATION_API §5).
- The portal type does not declare it (`API:1819–1842`) and nothing renders it.
- Prod: **0** users have it set today (deployed today; `null` = never chosen / device language). So the filter would read "All / English (never chose) / हिन्दी …".
- **Flag:** backend `language=` param (not supported today).
- Content language per user is **not persisted**. `user_profiles.medium` exists but is `ENGLISH` for all 16,244. Flag only.

**Dimension 4 · Push device** (rename of "Platform").
- Values: iOS / Android / None.
- Meaning: has an active push token on that OS (`US:119–123`).
- Drop "Web" (0 tokens).
- Add "None" (5,587 cannot be pushed). This needs a backend param (flag), or keep it out.
- Backend `platform=` ✅ exists.
- **The scan path must stop re-implementing it.** Minimal portal-side fix: put `platform` into `_sme_users_corpus_params` so upstream narrows by token and the local re-filter is skipped. The fix is in their code, so this is a flag for the owner.

**Joined (keep):** IST from/to, as today. **Flag:** backend `createdFrom` / `createdTo` params. With those, plus `access=`, the portal never needs the 163-call scan for this screen.

**Search (keep).**

**Minimal-change summary:**

| Status | Items |
|---|---|
| Backend already supports | `search`, `exam`, `onboarded`, `platform`, `status=SUSPENDED`, `premium=true(&exam=)` |
| Needs a backend param (flag only) | `access` (trial / trial_over / paid / comped / lapsed, exam-aware), `createdFrom` / `createdTo`, `language`, and push-device "none" |
| Portal-only meanwhile | relabel + merge the four selects into one "Access" select mapped to the existing params / local row fields; put `platform` into the corpus params |

### B.2 Plain-language test, per table element

| Element | One sentence a support person can say today | Bucket | Minimal change |
|---|---|---|---|
| Header "16243 users" | "16,243 accounts exist" (incl. 17 staff and 5,020 who never finished sign-up). Not "users" | **COMPACT** | "**16,227 people signed up · 11,207 finished onboarding** (17 staff hidden)". Every number is computable today from `user_auth` + `user_profiles` |
| C1 User + exam chip | "Asha, studying UPSC" | **KEEP** | chip "UPSC (default)" when `activeExamId` is null, so "never chose" is visible |
| C2 Premium column | Works for trial rows ("on trial, 3 days left"). Fails for APPSC day-0 rows ("Trial Ended" on the day they joined) and for never-onboarded users ("Trial Ended") | **COMPACT** → rename **Access** | One phrase per row, exam-aware: "**On UPSC trial · 3 days left**" / "**Paid UPSC until 3 Dec (Apple)**" / "**Comped APPSC until 3 Dec**" / "**No trial — APPSC has none**" / "**Never finished sign-up**" / "**Lapsed — UPSC ended 12 Aug**". All from fields already on the row (`premiumState`, `trialDaysLeft`, `entitlements[].{status,source,expiresAt}`, `activeExamId`, `onboardingCompleted`) |
| C3 Status | "ACTIVE" explains nothing (7,993 ACTIVE users have no access) | **RAW / NOISE** → drop | Keep only a red "Deactivated" tag when SUSPENDED |
| C4 Provider | "Signed in with Google": true, but it reads like a device | **COMPACT** | Fold into C1 as a small "Google / Apple sign-in" hint, or drop. Never place it next to "Platform" |
| C5 Joined | "Joined 3 Nov" | **KEEP** | Optionally add "**last opened the app 2 h ago**" (`max(sessions."lastAccessedAt")`, computable today, full history; the list row does not carry it → **flag** a backend field) |
| Drawer "Premium state / Exam mode / Trial / Entitlements" | "On UPSC trial till 6 Nov; holds nothing" | **KEEP** (S03 card 2/3 rules apply) | Empty-state copy should say "tried to pay" when a mandate exists (S03 M11) |
| Coverage banner | "This count is a floor: N of M rows read" | **KEEP AS IS** | Also apply it to Export (today silent) |
| Cohort send hint "Trial: … length set per exam" | **False** today (14-day rule) | **Fix (backend)** | Make `resolveSegment` use the same per-exam `getTrialEnd` as the list (flag), or relabel to "signed up in the last 14 days, never paid" |

**Derived metrics proposed for the list (computable today?):**
- **Last opened the app** = `max(sessions."lastAccessedAt")`. Yes, persisted, full history. Flag: not in the list row.
- **Days active, last 30** = `count(DISTINCT day)` from `api_usage_daily`. Yes, 120-day rollup. Flag: not in the row.
- **Paid vs comped** = `entitlements[].source`. Already on the row; nothing to persist.
- **Tried to pay** = `subscriptions` with `first_charge_granted_at IS NULL`, plus `orders` CREATED / FAILED. Persisted; **no SME route reads `subscriptions`** (S03). Flag.

### B.3 The six shareholder questions on this screen

| Q | Answered? | Missing | Persists? |
|---|---|---|---|
| (1) Feature use + repeat rate | **No.** A directory, no usage column | "last opened", "days active" | yes (`sessions`, `api_usage_daily`) |
| (2) Per-exam + per-language funnels | **Exam partially** (mode + holds work; Premium state is not exam-aware for non-X-mode users). **Language no** (no filter, no column) | `access` per exam; `language=` | yes (`user_exam_entitlements`, `user_profiles.ui_language`; 0 set today) |
| (3) Where buyers stop in checkout | **No.** No "tried to pay" access value | Access value "Tried to pay, never charged" (47 users / 61 mandates per S03 E14) | yes (`subscriptions`, `orders`) |
| (4) Question quality | n/a | — | — |
| (5) Notification outcomes | **No.** Sends only; and the Trial cohort over-reaches 5.9× | tapped / read per send | yes (`notification_dispatch.tapped_at`) |
| (6) Support view of one user | **Doorway only** (row → profile, S03). The drawer is accurate on holdings | mandates (S03) | yes |

---

## C. GAP (documented fields and params not forwarded or rendered)

| Item (doc) | Backend | Proxy | UI | Status |
|---|---|---|---|---|
| `premium=true&exam=` per-exam hold filter (SME_PORTAL_API §1.1 L45) | `US:93–106` | **never forwarded** (M:4446–4450); re-implemented by full scan | "Holds X" (`AU:1416`) | proxied-only (result equals truth) |
| `uiLanguage` on every row (SME_LOCALIZATION_API §5) | `US:771` | passthrough | **not in `SmeUserSummary` (`API:1819–1842`), not rendered, not exported** | proxied-only |
| `entitlements[].isTrial`, `accessTier` | `entitlement.service.ts:204–212` | passthrough | not on list or drawer (C2 row 8) | proxied-only |
| `trialExamId` | ✔ | ✔ | drawer only (`AU:1047–1057`), export ✔ | rendered (drawer) |
| `phoneVerified`, `role`, `lastLoginAt` | ✔ | ✔ | phoneVerified in drawer ✓; role absent; lastLoginAt export only | partial |
| `whatsappExamIds` / `whatsappExamsEffective` (detail) | `US:184–200` | ✔ | absent (drawer and profile) | proxied-only |
| Export columns | — | M:4616–4618 | lacks `uiLanguage`, `trialEndsAt` / `trialDaysLeft`, `phoneVerified`, entitlement `isTrial` / `accessTier`, push platforms | portal construct, incomplete |
| `status=ONBOARDING` | rows carry it; **DTO enum omits it** (`DTO:17–24`); doc enum omits it (§1.1 L43) | — | not selectable (`AU:84–86`) | absent (5,020 users) |
| `platform=web` | enum allows it | ✔ | option offered | dead (0 web tokens) |
| Stale UI copy | — | — | Cohort "Trial" hint "length is set per exam" (`AU:624`); Premium column header "Premium" over a funnel label; header "users" | false / ambiguous |
| Portal docstring M:4470–4473 "premium=true → 10 rows" | — | — | — | stale (25 today); harmless |

---

## D. Contradictions (reported, not papered over)

1. **The brief / SHARED_RULES B-addendum-2 says the live Status options are "ACTIVE/ONBOARDING/UNSUBSCRIBED/SUBSCRIBED/SUSPENDED".**
   - Live snapshot L140–145 and `AU:84–86` show ACTIVE, INACTIVE, SUSPENDED, LOCKED, SUBSCRIBED, UNSUBSCRIBED. **ONBOARDING is not an option.**
   - It appears only as a row value (5 of 20 rows).
   - INACTIVE and LOCKED match 0 users.
2. **SME_NOTIFICATIONS_API.md L218**, on the `trial` segment: "`trial_ends_at` still in the future — the trial length configured for the exam… **not a fixed 14 days**. Rows with no `trial_ends_at` (pre-dating the column) fall back…".
   - In code, `trial_ends_at` is NULL for **every freshly-created user** (`PCU:19–20`), and only an SME extension writes it (`US:482–488`; 4 ext users ever).
   - So the "legacy fallback" (`US:645, 663–673`) is the rule for everyone: a fixed 14 days. Prod: 1,609 vs 274.
   - The UI repeats the doc's claim (`AU:624`). C2 marked SME_NOTIFICATIONS row 4 "RENDERED" and missed this.
3. **SME_PORTAL_API §1.1 L45**: `premium` "boolean — `true` = currently premium".
   - `premium=false` is silently ignored (`US:92`, only `=== true`). The portal knows (M:4470–4471); the doc does not say.
4. **C2 SME_TRIAL_EXTENSION row 5** ("400 for SUSPENDED/LOCKED/ONBOARDING/INACTIVE … `TRIAL_EXTENSION_BLOCKED_STATUSES` → RENDERED").
   - The UI list is `['SUSPENDED','LOCKED','INACTIVE']` (`AU:112`), with no ONBOARDING, so the button is enabled for 5,020 users our side refuses.
5. **AUDIT §1.2 row** (`/app-users` → `GET /sme/users` passthrough "unless premium, premiumState, createdFrom/To").
   - The code is correct, but it omits that **Export is always a full scan** and that **the Platform filter changes meaning on that path**. (C2 #13 and AUDIT §5C mention the meaning change; neither quantifies it as "always 0 rows / empty export".)
6. **`DTO:15–16` comment** says "identity-only statuses like ONBOARDING are not settable here". The same enum also governs the **list filter** (`DTO:52–55`), so ONBOARDING is not *filterable* either. The edit modal then 400s on name-only edits of ONBOARDING users (§A.1c).
7. **SHARED_RULES says "No prod/staging SQL in this pass"; this lane's brief allows prod read-only SQL.** I followed the brief: SELECT only, ids and counts only.
8. **Premium state doc** (SME_PORTAL_API §1.1 L215–218) lists "never onboarded" among the states where `trialEndsAt` is null. That is true, but it implies the label distinguishes them. In fact 4,850 never-onboarded users derive **"Trial Ended"** (`PCU:322–323`). Only ONBOARDING users still inside the window derive "Downloaded" (170).

---

## E. SQL run against production (read-only), with results

Every query lives in the scratchpad (`s04_q1.sql` … `s04_q6.sql`; the shared CTE is `s04_base.sql`). The CTE reproduces `formatUserSummary` exactly:
- **Trial days.** Per `activeExamId ‖ upsc-cse`, from `exams.trial_policy`:
  - UPSC: 3 days if created ≥ 2026-09-25 04:14:27.647, else 14.
  - APPSC: 0 days if created ≥ 2026-09-16 17:14:15, else 14.
- **Trial end.** `trial_end = coalesce(trial_ends_at, createdAt + days)`.
- **Paid mirror.** `paid_mirror = SUBSCRIBED ∧ (premium_expires_at null ∨ > now)`.
- **In trial.** `in_trial = ACTIVE ∧ now < trial_end`.
- **Premium state.** The `pstate` derivation follows `PCU:313–324`.
- **Live entitlement.** `live_* = user_exam_entitlements` with `revoked_at IS NULL ∧ (expires_at IS NULL ∨ > now)`.
- **Internal accounts.** Flagged by `email ILIKE '%@stanzasoft.%' OR '%@prepmonkey.%' OR '%@designasylum.in'`.
- **Schema check.** `information_schema.columns` was checked first for `user_auth`, `user_profiles`, `device_tokens`, `user_exam_entitlements`, `exams`.

| # | Query | Result |
|---|---|---|
| E1 | `status` buckets `ROLLUP`, all / internal / ext (`s04_q1`) | total **16,244 / 17 / 16,227**. ACTIVE 8,267 · ONBOARDING 5,023 (ext 5,020) · UNSUBSCRIBED 2,927 (2,918) · SUBSCRIBED 25 (20) · SUSPENDED 2 · INACTIVE 0 · LOCKED 0 |
| E2 | `pstate × status` with live-entitlement counts (`s04_q2`) | Premium 25 / 20 ext (24 / 20 hold LIVE) · Trial 274 (all ACTIVE) · **Trial Ended 15,758 / 15,751 ext = ACTIVE 7,993 + ONBOARDING 4,850 + UNSUBSCRIBED 2,907 + SUSPENDED 1** · Churned 17 / 12 · Downloaded 170 (all ONBOARDING) |
| E3 | per-filter, per-path counts (`s04_q3`), all / ext | SCAN premium=true 299 / **294** · upstream premium=true 25 / 20 · SCAN premium=false 15,945 / 15,933 · LIVE any 24 / **20** · LIVE non-MANUAL 17 / **16** · trial 274 · FAST ios 4,898 / **4,893** · android 5,795 / 5,786 · web **0** · both 42 / 39 · no token 5,593 / 5,587 · **SCAN ios (provider/source='ios') 0** · provider=apple 1,685 · exam=UPSC 15,045 / **15,031** (literal upsc-cse 3,390 / **3,383**) · exam=APPSC 1,199 / 1,196 · holds LIVE UPSC 19 / **15** · holds LIVE APPSC 10 / **7** (4 / 1 not in APPSC mode) · APPSC-mode ∧ isPremium 6 · onboarded=true 11,219 / 11,207 · onboarded=false (both paths) 5,025 / 5,020 · no-profile rows 0 · ONBOARDING ∧ onboarded 0 · **ui_language set 0** |
| E4 | ext by exam mode × pstate × status (`s04_q4`) | APPSC: 953 ACTIVE Trial Ended (**81** created < 3 d ago, UPSC window still open), 232 UNSUBSCRIBED, 6 Premium, 4 Churned, **0 Trial** · NULL-mode: 4,732 ACTIVE + 4,849 ONBOARDING + 1,884 UNSUBSCRIBED Trial Ended, 169 Downloaded, 8 Premium, 0 Trial · UPSC-chosen: 274 Trial, 2,308 ACTIVE + 791 UNSUBSCRIBED Trial Ended, 6 Premium |
| E4b | ext entitlement rows by exam × source (LIVE / all / users) | APPSC APPLE 4/6/6, MANUAL 2/3/3, RAZORPAY 1/1/1 · UPSC APPLE 5/6/6, MANUAL 4/7/7, RAZORPAY 6/7/7 |
| E5 | segment-vs-list (`s04_q5`), ext | `trial` segment (14-day rule) **1,609**, of which list Trial 274, list **Trial Ended 1,335**; list Trial ∧ not in segment 0 · SME-set `trial_ends_at` 4 (1 in future) · Churned 12 (3 only-MANUAL) · Premium whose LIVE rows are all MANUAL **4** · signups today IST 104 |
| E6 | platform combos (`s04_q6`), ext | isPremium ∧ iOS **151** · isPremium ∧ Android 117 · LIVE ∧ iOS 14 · LIVE ∧ Android 7 · joined last 7 IST days ∧ iOS **419** · Apple sign-in ∧ Android token 6 · Google sign-in ∧ iOS token **3,419** · Trial Ended ∧ ONBOARDING 4,850 · Trial Ended ∧ ACTIVE 7,993 |
| E7 | `exams` (id, access_tier, trial_days, trial_policy) | upsc-cse paid 3 (policy 14 → 3 from 2026-09-25) · appsc-group-1 paid 0 (14 → 0 from 2026-09-16) |
| E8 | distinct `provider` / `login_provider` / `subscription_source`; `user_profiles.medium` | provider google 14,559 / apple 1,685; login_provider all null; subscription_source APPLE 11 / MANUAL 9 / RAZORPAY 9 / null 16,215; medium ENGLISH × 16,244 |

**Reconciliation**
- Header "16243 users" (capture) vs `count(*)` 16,244 (query time): **+1 signup** in between.
- Ext count is 16,227.
- The live page-1 composition (13 UPSC Trial 3d / 5 ONBOARDING Downloaded / 2 APPSC ACTIVE Trial Ended, all joined 03 Nov) is consistent with E4.

---

## F. Processes and files

- Processes: no background process or server was started. Each `npx ts-node -T scripts/staging-db.ts --prod …` call (8) and the one `python3 -I` snapshot tabulation ran in the foreground and exited. **No pids remain.** No browser was used; the captures were the coordinator's.
- Files written:
  - this report, `/private/tmp/claude-502/-Users-yashwanth-Desktop-projects/b0942abe-5468-460e-92b6-db352e6a3606/scratchpad/phase1/S04_app_users_list.md`;
  - SQL inputs `/private/tmp/claude-502/-Users-yashwanth-Desktop-projects/b0942abe-5468-460e-92b6-db352e6a3606/scratchpad/s04_q1.sql` … `s04_q6.sql` and `s04_base.sql`.
- No edits, commits, deploys or mutations. No `.env*` read. No `git fetch/pull/checkout`.
