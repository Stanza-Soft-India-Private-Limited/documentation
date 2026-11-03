# SME Portal — Management API Guide

> Changed 2026-09-21 — exam dimension on users, orders, offers, banners; two BREAKING calls (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).
> Changed 2026-09-21 — per-exam broadcast, validated segment exam (+ fix), survey targetExams, feedback ?exam= (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).
> Changed 2026-11-03 — lane B1 "money truth": internal accounts excluded by default (`includeInternal`), `isInternal` / `isComp` / `isSandbox` row flags, IST calendar-day date filters, payment attempts (incl. UPI AutoPay mandates), per-exam money (see [WHAT_CHANGED_2026-11.md §B1](./WHAT_CHANGED_2026-11.md#b1--money-truth-contract-backend-lane-b1-2026-11-03)).
> Changed 2026-11-03 (lane B6) — `GET /sme/users?platform=none` (no active push device); `platform` validated (see [WHAT_CHANGED_2026-11.md §B6](./WHAT_CHANGED_2026-11.md#b6--follow-ups-surfaced-by-the-portal-lanes-backend-lane-b6-2026-11-03)).

End-to-end reference for the **SME portal** to manage the PrepMonkey app's users,
transactions, blogs, and notifications. This is the management/admin surface of the
app exposed server-to-server.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `BASE_URL` = `https://app.stanzasoft.ai`)
**Authentication:** `x-api-key: <API_KEY_SECRET>` header on **every** `/sme/*` request.
**Note:** All endpoints use the `/api/v1/` global prefix. Swagger UI at `/api/docs`
(group **SME**) documents these live with the `x-api-key` control.

```
GET /api/v1/sme/users
x-api-key: <API_KEY_SECRET>
```

- Missing/invalid key → **401** `{ "message": "Invalid or missing API key" }`.
- Money amounts in **responses** are in **rupees**; the `amount` you optionally pass
  to a premium grant is in **paise** (matches how the app stores prices).
- Pagination: list endpoints take `page` (default 1) + `limit` (default 20, max 100)
  and return `{ data, total, page, limit, hasMore }`.
- **Internal accounts (2026-11, B1).** `/sme/users`, `/sme/transactions`, `/sme/orders`,
  `/sme/webhooks` and `/sme/payment-attempts` EXCLUDE internal accounts from rows and
  `total` by default: emails at stanzasoft.com / stanzasoft.ai / prepmonkey.com /
  prepmonkey.ai / designasylum.in, the store-review demo phone logins, and owners of an App
  Store sandbox order. `includeInternal=1` includes them; every row carries `isInternal`.
  A single user's page (`/sme/users/:id`, `/snapshot`) is never filtered.
- **Dates (2026-11, B1).** `startDate`/`endDate` (and `from`/`to`) given as `YYYY-MM-DD`
  are **IST calendar days**: start inclusive from 00:00 IST, end = the whole day.
  `startDate = endDate` is one day. A full ISO timestamp is taken literally; garbage → 400.
- **No account-wide "premium" (2026-11 ruling).** Paid / comped / revenue figures are per
  exam; account-level `isPremium` / `premiumState` / `premiumExpiresAt` /
  `status=SUBSCRIBED` / trial describe the **UPSC-CSE** exam (legacy default; also matches
  users with no active exam). Per-exam truth is `entitlements[]`.

---

## 1. User Management — `/sme/users`

Users originate from Cognito OAuth (Google/Apple) + onboarding, so there is **no
"create user"** endpoint. SME can list, view, edit (name/status), deactivate, delete,
grant/revoke premium, extend a free trial, and notify.

### 1.1 List users
```
GET /sme/users?page=1&limit=20&search=&status=&exam=&access=&language=&createdFrom=&createdTo=&onboarded=&platform=
```
> **2026-11 (lane B3) — the filter model is Exam → Access → Language** (AUDIT_2026-11 §5B
> ruling b; owner ruling 2026-11-03: the account-wide word "premium" is retired).
> `access=` is the one access axis and is **always exam-scoped**; `premium=` and
> `premiumState=` stay accepted as **LEGACY** (account-wide, UPSC-era state) — use
> `access=` + `exam=` instead.
| Query | Type | Description |
|---|---|---|
| page | number | default 1 |
| limit | number | default 20, max 100 |
| search | string | matches email / username / name / phone (case-insensitive) |
| status | enum | ACTIVE, INACTIVE, SUSPENDED, LOCKED, **ONBOARDING** (2026-11), SUBSCRIBED, UNSUBSCRIBED. A stale lifecycle mirror — for access use `access=`; keep `status=SUSPENDED` as the "deactivated" modifier. |
| **exam** | slug | Filter by the user's **active exam mode** (`user_profiles.active_exam_id`). `exam=upsc-cse` **also matches users who have never picked an exam** (`active_exam_id` NULL) — the same rule push segments use, and the only reason the default exam returns anybody. **400 on an unknown slug**, never a silently empty page. |
| **access** | enum | `paid` \| `comped` \| `not_onboarded` \| `trial` \| `lapsed` \| `trial_over` — see **§1.1a**. What the person **holds in `exam`**, over every user whatever exam they have selected (with `access`, `exam` does not apply the selected-exam rule above). **Without `exam` it is evaluated against `upsc-cse`** and the response says `examDefaulted: true`. 400 on any other value. |
| **language** | string | `user_profiles.ui_language`, exact code (`hi`, `ta`, `pt-br`; lower-cased). `none` = never chosen (NULL or no profile) — NOT coerced to English: NULL means the device language. |
| **createdFrom** / **createdTo** | date | Joined between. A bare `YYYY-MM-DD` is an **IST day**: `createdFrom` from that day's IST midnight (inclusive), `createdTo` through the end of that IST day (bound = next IST midnight, exclusive), so `createdFrom=createdTo=D` is exactly day D. An ISO datetime is used as an inclusive instant. 400 on an impossible date (`2026-02-31`) or when from is after to. Removes the portal's 163-call full scan for "Joined". |
| premium | boolean | **LEGACY** (account-wide, UPSC-era). `true` = currently premium. **Without `exam`:** the person-level mirror (`user_auth` SUBSCRIBED and not expired). **With `exam`:** a live, un-revoked `user_exam_entitlements` row for **that** exam (NULL expiry = perpetual). `false` (2026-11) is the exact complement — until then it was **silently ignored**. Prefer `access=`. |
| premiumState | enum | **LEGACY** (2026-11; was a 400). `Premium` \| `Trial` \| `Trial Ended` \| `Churned` \| `Downloaded` — returns exactly the rows whose `premiumState` shows that label (same derivation, person-level trial length). Prefer `access=` + `exam=`. |
| onboarded | boolean | filter by onboarding completion |
| platform | enum | `ios` \| `android` \| `web` = has an **active** push token on that platform; **`none`** (2026-11, B6.4) = has **no active** push token on any platform — the people a push cannot reach (never granted permission, uninstalled, or FCM deactivated the token; inactive tokens do not count). Any other value is a **400** (it used to match nothing and return an empty page — `none` silently returned 0). |

#### 1.1a The Access axis (`access=`, 2026-11)

Every user of the scope is in **exactly one** value (the six counts add up to the list
total; proven on staging):

| Value | Meaning (scope = `exam`'s rows and trial, or `upsc-cse` when no exam is sent; over every user) |
|---|---|
| `paid` | holds a **live** entitlement whose source is not MANUAL (APPLE / RAZORPAY) |
| `comped` | holds a live **MANUAL** entitlement and no live paid one (staff comp) |
| `not_onboarded` | never finished onboarding (no profile, or `onboarding_completed = false`) and holds nothing live |
| `trial` | onboarded, holds nothing live, and is **inside the free-trial window** |
| `lapsed` | onboarded, has entitlement rows but none live, not in the trial window |
| `trial_over` | onboarded, never held an entitlement, not in the trial window — includes "this exam has no trial" (0-day exams) |

- **live** = `revoked_at IS NULL AND (expires_at IS NULL OR expires_at > now)`.
- **in the trial window** = the gate's own rule: status ACTIVE or SUBSCRIBED (a purchase
  in one exam does not end the trial in another) AND now < trial end, where an SME-set
  `trial_ends_at` wins, else `created_at` + **`exam`'s** trial length (`exams.trial_policy`
  entry in force at signup; an exam not in the catalogue = 14 days).
- **It is the HOLDING, not the selection (B3.1c, coordinator ruling).** With `access=`,
  `exam` means "in this exam" for every user — the selected-exam rule (`active_exam_id`,
  NULL ≡ upsc-cse) is **not** applied. An APPSC buyer browsing in UPSC mode is in
  `exam=appsc-group-1&access=paid` (likewise `comped` / `lapsed`). The six values therefore
  partition the **whole** user base once per exam, and each row's `access[]` line for that
  exam carries the same state. The plain list **without** `access=` keeps the
  selected-exam rule, unchanged.
- **Backfill-stable (owner rule 2026-11-03).** The axis reads `user_exam_entitlements` +
  `active_exam_id` + the trial clock only. The user_auth mirror (`status=SUBSCRIBED`,
  `premium_expires_at`, `subscription_source`) enters through ONE mapping: a SUBSCRIBED
  user with **no entitlement row in any exam** reads as holding one virtual `upsc-cse`
  row (`source = COALESCE(subscription_source,'MANUAL')`, `expires_at =
  premium_expires_at`) — the shape the 2026-09-08 migration wrote. A later backfill that
  writes exactly that row (same predicate) changes no answer.

**Response additions (2026-11):** the list body always carries `examDefaulted`
(boolean — `true` only when `access` was sent without `exam`), and **every row** carries
`access[]` — one line per active exam (plus any exam the user holds a row in):
```json
"access": [
  { "examId": "upsc-cse", "state": "trial", "until": "2026-11-06T09:12:00.000Z", "source": null },
  { "examId": "appsc-group-1", "state": "paid", "until": "2027-11-03T00:00:00.000Z", "source": "APPLE" }
]
```
`state` uses the `access=` vocabulary (same precedence, same legacy mapping, the trial
evaluated with **that** exam's length). `until`: paid/comped = when that access ends
(null = perpetual); trial = trial end; lapsed = when the last entitlement ended;
trial_over = when the trial ended (the signup instant on a 0-day exam); not_onboarded =
null. `source`: the entitlement source for paid/comped/lapsed, else null.
`isPremium` / `premiumState` remain on the row for compatibility — they are the
account-wide legacy view.

> **Exam is two axes here, and they answer different questions.** `exam=` alone filters
> on who the user **is** (`active_exam_id`). `exam=` + `premium=true` filters on what
> they **hold** (an entitlement row). A user who studies UPSC but bought APPSC appears
> under `exam=upsc-cse` and under `exam=appsc-group-1&premium=true` — both are correct.
> With `premium=true&exam=`, `user_auth.status` is deliberately **not** constrained: the
> entitlement rows are the authority and the mirror may lag a minute behind.

**Response 200:**
<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/users?exam=appsc-group-1&limit=2` — verbatim, 2 of 4 rows:

```json
{
  "data": [
    {
      "id": "2f9b6d14-08ac-4e57-b3d6-7c1e29f4a860",
      "email": "aspirant-3@example.com",
      "username": "aspirant_s7u8v9",
      "name": "S. Rao",
      "phoneNumber": null,
      "phoneVerified": false,
      "provider": null,
      "role": "USER",
      "status": "ACTIVE",
      "isPremium": true,
      "premiumState": "Trial Ended",
      "premiumExpiresAt": null,
      "subscriptionSource": null,
      "onboardingCompleted": true,
      "lastLoginAt": null,
      "createdAt": "2026-09-16T08:52:37.680Z",
      "trialEndsAt": null,
      "trialDaysLeft": null,
      "trialExamId": "appsc-group-1",
      "activeExamId": "appsc-group-1",
      "entitlements": []
    },
    {
      "id": "1c7e4a90-5d33-4b18-9f02-6ab8c4e51d77",
      "email": "aspirant-2@example.com",
      "username": "aspirant_r4s5t6",
      "name": "R. Iyer",
      "phoneNumber": "+91XXXXXXXXXX",
      "phoneVerified": true,
      "provider": "google",
      "role": "USER",
      "status": "ACTIVE",
      "isPremium": true,
      "premiumState": "Trial",
      "premiumExpiresAt": null,
      "subscriptionSource": null,
      "onboardingCompleted": true,
      "lastLoginAt": "2026-09-16T07:25:13.888Z",
      "createdAt": "2026-09-16T06:12:21.958Z",
      "trialEndsAt": "2026-09-30T06:12:21.958Z",
      "trialDaysLeft": 9,
      "trialExamId": "appsc-group-1",
      "activeExamId": "appsc-group-1",
      "entitlements": []
    }
  ],
  "total": 4,
  "page": 1,
  "limit": 2,
  "hasMore": true
}
```

> ℹ️ **Captured 2026-09-21, before lanes B1/B3.** Every row now also carries `isInternal`,
> `uiLanguage` (the app UI language, `null` = never chosen — filter with `language=`) and
> `access[]` (§1.1a), and the page carries `examDefaulted`. The captured keys are unchanged.

> ⚠️ **Row 1 above is a PRE-FIX capture. `isPremium: true` next to `premiumState: "Trial
> Ended"` and `entitlements: []` is the bug this release fixes** — it was taken from
> staging on 2026-09-21 minutes before the fix landed, and is kept because it is the
> clearest illustration of what changed. **On the released build that row returns
> `isPremium: false`.** Every other value in the capture is unaffected.
>
> **`isPremium` now honours the exam's own trial length**, so it agrees with
> `premiumState` and with `entitlements[]` on every row: `isPremiumUser()` is given the
> same per-exam `trialDays` that `derivePremiumState()` already used, and the same fix is
> threaded through the snapshot, the payments path, the notification audience and the
> `tier_premium` FCM topic. A `Trial Ended` user on a 0-day-trial exam is no longer
> premium anywhere.
>
> *Before 2026-09-21*, `isPremiumUser()` took no `trialDays` argument: for an `ACTIVE`
> user with `trial_ends_at` NULL it fell back to `created_at + 14 days` regardless of the
> exam, so on APPSC Group 1 (`trialDays: 0`) a lapsed trialist read as premium — and was
> pushed as `tier_premium` — for the first 14 days after signup.
>
> **The portal guidance is unchanged, and still worth following:** use `premiumState` for
> the funnel label and `entitlements[]` for "what does this person actually hold in exam
> X". `isPremium` is the person-level `user_auth` mirror: `true` for a `SUBSCRIBED` user
> whose `premiumExpiresAt` is unset or in the future, **and also for an `ACTIVE` user
> inside the trial window** of their own active exam (`upsc-cse` when unset) — it does not
> look at `entitlements[]`. Read it as the UPSC-CSE-era state (top of this doc), never as a
> per-exam badge.

A row that *does* hold something — from `GET /sme/users?exam=appsc-group-1&premium=true`,
which returns `total: 1` against the same 4-user population:

```json
{
  "data": [
    {
      "id": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
      "email": "aspirant@example.com",
      "username": "aspirant_a1b2c3",
      "name": "A. Sharma",
      "phoneNumber": "+91XXXXXXXXXX",
      "phoneVerified": true,
      "provider": "google",
      "role": "USER",
      "status": "SUBSCRIBED",
      "isPremium": true,
      "premiumState": "Premium",
      "premiumExpiresAt": "2026-10-16T12:09:32.616Z",
      "subscriptionSource": "RAZORPAY",
      "onboardingCompleted": true,
      "lastLoginAt": "2026-09-16T16:29:40.735Z",
      "createdAt": "2026-09-09T11:49:43.440Z",
      "trialEndsAt": null,
      "trialDaysLeft": null,
      "trialExamId": "appsc-group-1",
      "activeExamId": "appsc-group-1",
      "entitlements": [
        {
          "examId": "appsc-group-1",
          "accessTier": "paid",
          "status": "LIVE",
          "source": "RAZORPAY",
          "expiresAt": "2026-10-16T12:09:32.616Z",
          "isTrial": false,
          "trialEndsAt": null
        }
      ]
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 20,
  "hasMore": false
}
```

An unknown slug is a 400, captured the same day:
`GET /sme/banners?exam=appsc-grp-1` →
`{"success":false,"message":"Unknown exam \"appsc-grp-1\". Create it via POST /sme/exams first.","error":"Bad Request","statusCode":400,…}`
— the same validator backs `?exam=` on `/sme/users`, `/sme/orders`, `/sme/transactions`
and the notification sends.

**The three exam fields on every row** (batch-loaded — one query for the whole page, not
one per row):

| Field | Meaning |
|---|---|
| `activeExamId` | **Who they are.** `user_profiles.active_exam_id`, **nullable** — `null` means "never picked", which is deliberately *not* coerced to `upsc-cse`: "never chose" and "chose UPSC" are different facts, and only the `exam=` filter conflates them. |
| `trialExamId` | Which exam's trial policy produced `trialEndsAt` (`activeExamId`, or `upsc-cse`). Present so a portal showing a 7-day window can say **which** exam decided that instead of looking like a bug. |
| `entitlements[]` | **What they hold.** One entry per `user_exam_entitlements` row: `{ examId, accessTier, status, source, expiresAt, isTrial, trialEndsAt }`. `status` ∈ `LIVE` \| `EXPIRED` \| `REVOKED` (REVOKED beats an expiry still in the future — that is what a refund means). `source` is **per row** (`APPLE` \| `RAZORPAY` \| `MANUAL`), not the person-level `subscriptionSource`. `expiresAt: null` = perpetual, not unknown. `isTrial` is true only when the row has stopped granting anything *and* the person-level trial still covers that exam. |

> **A trial-only user has `entitlements: []`.** A trial creates no row. Read
> `premiumState` / `trialEndsAt` for them — an empty array is not "holds nothing broken",
> it is the normal shape for someone who has never bought.

> **`total` is cross-exam unless you send `?exam=`.** The count is computed from the same
> `where` as the page, so a headline built from an unfiltered call is every exam's users
> added together. Send `?exam=` whenever the number is going to be labelled with an exam.

`premiumState` is a derived funnel label: `Premium` / `Trial` / `Trial Ended` /
`Churned` / `Downloaded` (computed from live subscription state, not the raw column).
`Trial` is also reachable for a previously-`Churned` user while an SME trial
extension is active (§1.8) — they fall back to `Churned` once the extension lapses.

`trialEndsAt` (ISO string) / `trialDaysLeft` (int, rounded up) are populated **only**
while the user is currently in an active trial — `status=ACTIVE` and now before the
trial end — whether that's the exam's own trial window or an SME-extended one. Both
are `null` for every other state (premium, trial ended, churned, never onboarded).
**The window length is per exam** (`trialExamId` names which exam decided it), so do
not hard-code 14 anywhere in the portal.

> **`isInternal`** (2026-11) is on every row. Internal rows are hidden unless
> `includeInternal=1` — a support search for a staff account needs it.

### 1.2 Get a user

> **2026-11:** also returns `isInternal` and **`paymentAttempts[]`** — the newest 20 payment
> attempts on every rail (Apple, Razorpay one-time, UPI AutoPay **mandate** from
> `subscriptions`, staff COMP), same row shape as §2.5. `recentOrders` is unchanged but
> cannot see a mandate that was never charged; render the Money block from
> `paymentAttempts`.
```
GET /sme/users/:id
```
**Response 200:** all summary fields (including `activeExamId`, `trialExamId`,
`entitlements[]`, `trialEndsAt` / `trialDaysLeft`, §1.1) plus:

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/users/0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30` — verbatim, with the §1.1 summary
fields collapsed into the first line (they are byte-identical to the `premium=true` row
above, `entitlements[]` included):

```json
{
  "…": "every §1.1 summary field, incl. activeExamId / trialExamId / entitlements[]",
  "trialExtended": false,
  "whatsappExamIds": null,
  "whatsappExamsEffective": ["appsc-group-1", "upsc-cse"],
  "profile": {
    "id": "4e8c1b26-7f95-4a03-b1d8-52c6e09af417",
    "userId": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
    "phone": "",
    "dateOfBirth": null,
    "attempt_year": 2027,
    "medium": "ENGLISH",
    "upsc_roll": null,
    "graduationYear": null,
    "graduationStream": null,
    "previousAttempts": 0,
    "targetYear": null,
    "optionalSubject": null,
    "profileCompletionPercentage": 15,
    "onboardingCompleted": true,
    "psychometricTestCompleted": false,
    "aspirantType": "FULL_TIME",
    "studyTimeAllocationHours": 0,
    "studyTimeAllocationMinutes": 0,
    "studyStartTime": "09:00",
    "studyEndTime": "09:00",
    "psychometricTestSkipped": true,
    "studyHoursPerDay": null,
    "longTermGoals": null,
    "shortTermGoals": null,
    "activeExamId": "appsc-group-1",
    "createdAt": "2026-09-09T11:49:43.440Z",
    "updatedAt": "2026-09-16T16:29:59.055Z"
  },
  "platforms": ["android", "ios"],
  "recentOrders": [
    {
      "id": "5b71d908-3c42-4f6b-a087-13e9d5c82b64",
      "amount": 499,
      "currency": "INR",
      "status": "PAID",
      "planType": "MONTHLY",
      "paymentSource": "RAZORPAY",
      "examId": "appsc-group-1",
      "isTest": false,
      "premiumGrantedAt": "2026-09-16T12:09:32.623Z",
      "createdAt": "2026-09-16T12:09:32.607Z",
      "latestPayment": {
        "id": "6c82ea19-4d53-4071-b198-24fae6d93c75",
        "status": "CAPTURED",
        "method": "card",
        "amount": 499
      }
    }
  ]
}
```

> **`profile.activeExamId` duplicates the top-level `activeExamId`** — same column, read
> twice because `profile` is the raw `UserProfile` row. They cannot disagree; prefer the
> top-level one so the portal does not depend on `profile` being present.
>
> **`profile` is `null` when the user has no `user_profiles` row** (`profile:
> user.userProfile ?? null` in the service) — do not index into it unguarded.
`recentOrders[].examId` is **which exam the money bought**, and it is **nullable** —
`null` is a real state, not missing data. Apple's payloads carry no exam, so an order
whose signed `productId` matches no `exam_plans` row is recorded *unresolved* rather
than charged to a guessed exam. Those rows are the `UNRESOLVED_EXAM` reconcile cases in
§2.3. `isTest: true` marks an App Store sandbox purchase — listed here on purpose (it is
this person's record) but never revenue.

`trialExtended` (boolean) is `true` when an SME trial extension (§1.8) is currently
in effect for this user — i.e. the underlying `trialEndsAt` column is set and still
in the future. This is distinct from `trialDaysLeft != null`, which is also true for
a user riding out their exam's ordinary (never-extended) trial window; `trialExtended`
tells the portal specifically "an SME granted extra time here."

`whatsappExamIds` / `whatsappExamsEffective` — which exams this person gets WhatsApp
(Wylto) updates for, the same pair `GET /user/profile/me` returns. `whatsappExamIds` is
what the user STORED from the app's Profile switches: `null` = never chosen, `[]` =
explicitly none. `whatsappExamsEffective` is what actually applies: the stored list, or —
when never chosen — their active exam (UPSC if unset) plus every exam with a LIVE
entitlement. Drives the `EXAM_UPSC` / `EXAM_APPSC` Wylto tags (when enabled). Read-only
here; push/feed audiences are unaffected.

`404` if the user doesn't exist.

### 1.3 Update a user (name / status only)
```
PATCH /sme/users/:id
{ "name": "New Name", "status": "ACTIVE" }
```
Both fields optional. `status` ∈ ACTIVE, INACTIVE, SUSPENDED, LOCKED, SUBSCRIBED,
UNSUBSCRIBED, and **ONBOARDING only unchanged** (2026-11): the portal's edit modal resends
the row's status with every name change, so an ONBOARDING user's name edit used to 400;
it now succeeds, while moving anyone INTO ONBOARDING is a 400 ("status ONBOARDING cannot
be set by SME"). Phone, email and role are read-only. Returns the updated user summary —
**the same shape as a §1.1 list row**, `activeExamId` / `trialExamId` / `entitlements[]`
included and freshly loaded. A name correction never blanks out what the user holds.

### 1.4 Deactivate (soft delete — reversible)
```
POST /sme/users/:id/deactivate
```
Sets `status=SUSPENDED` and revokes all active sessions. **Response:**
`{ "success": true, "status": "SUSPENDED" }`. Reverse by `PATCH`-ing status back to ACTIVE.

### 1.5 Delete (hard delete — irreversible)
```
DELETE /sme/users/:id
```
Permanently purges the user: **Postgres cascade** (profile, orders, payments, content,
attempts, device tokens, notifications…) **+ Neo4j** node and relationships. **Cannot be
undone.** Response: `{ "success": true, "message": "Account deleted successfully" }`.

### 1.6 Grant premium
```
POST /sme/users/:id/premium
{ "plan": "MONTHLY", "amount": 59900, "reference": "rzp_pay_XXXX", "examId": "upsc-cse" }
```
| Field | Required | Description |
|---|---|---|
| plan | yes | `MONTHLY` (30d) or `ANNUAL` (365d) |
| amount | no | paise to record on the transaction; defaults to **that exam's** plan price |
| reference | no | external reference (e.g. the real Razorpay payment id being reconciled), stored on the order/payment notes + audit log |
| examId | no | **Which exam to grant.** Defaults `upsc-cse`. Each exam is sold separately, so a comp for one grants nothing in another. |

Creates an **auditable** `Order(source=MANUAL, status=PAID)` + `Payment(CAPTURED)`, grants a
`user_exam_entitlement` for that exam (**extending** any live one, otherwise starting now), and
refreshes the person-level mirror on `user_auth`. Syncs Wylto CRM. Use this to **fix a payment
that was taken but premium never set** (or to comp a user).

**Response 200:**
```json
{
  "success": true, "plan": "MONTHLY", "examId": "upsc-cse",
  "premiumExpiresAt": "2026-07-16T…",
  "orderId": "ord_…", "paymentId": "pay_…", "amount": 599
}
```

### 1.7 Revoke premium

> ## 🔴 BREAKING (2026-09-21) — `?exam=` is now REQUIRED
>
> The bare `DELETE /sme/users/:id/premium` used to revoke **every** exam. It is now a
> **400**, verbatim:
>
> ```
> exam is required: pass ?exam=<examId> to revoke one exam, or ?exam=all to revoke every exam.
> ```
>
> **Why there is no default.** Exams are sold independently. The old bare call, used to
> refund one ₹499 APPSC purchase, also destroyed the same person's ₹4,999 UPSC
> subscription. There is no safe default, so there is no default: pass a slug, or pass
> `all` and mean it.
>
> **Portal action:** every existing "Revoke premium" button must now send an exam. If the
> UI cannot yet ask which, send `?exam=all` — that is the explicit opt-in to the old
> behaviour — but prefer a picker sourced from the user's `entitlements[]` (§1.1).

```
DELETE /sme/users/:id/premium?exam=appsc-group-1   # just that one
DELETE /sme/users/:id/premium?exam=all             # every exam (explicit opt-in)
DELETE /sme/users/:id/premium                      # 400 — see the box above
```
| Query | Required | Description |
|---|---|---|
| exam | **yes** | An existing exam slug, **or** the literal `all`. An unknown slug (anything but `all`) is a **400** — `user_exam_entitlements.exam_id` has no FK, so a typo would revoke nothing while the portal reported success. |

Revokes the matching entitlement row(s), then **recomputes `user_auth` from what
survives** — so a user who still holds another exam correctly **stays** `SUBSCRIBED`,
and the audit row records the status that actually resulted rather than asserting
`UNSUBSCRIBED`. With `?exam=all` everything goes and `premiumExpiresAt` is force-expired
to now (kept non-null so Wylto still reads *Churned* rather than *never paid*).

**The missing-`exam` 400, captured:**
<!-- captured from staging 2026-09-21, backend f6329e6 -->
`DELETE /sme/users/0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30/premium` (no query string) →
**400**:

```json
{
  "success": false,
  "message": "exam is required: pass ?exam=<examId> to revoke one exam, or ?exam=all to revoke every exam.",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:37:37.090Z",
  "path": "/api/v1/sme/users/0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30/premium",
  "method": "DELETE"
}
```

Those seven keys — `success` · `message` · `error` · `statusCode` · `timestamp` · `path`
· `method` — are the **standard error envelope on every 4xx in this doc**. Read
`message` for the human text; it is the only field that varies usefully.

**Response 200** — shape from code; **not captured**, because a successful revoke is a
destructive write and the only premium row on staging is a live one we were not willing
to burn:

```json
{ "success": true, "examId": "appsc-group-1", "expiredAt": "2026-09-21T10:30:00.000Z" }
```
`examId` echoes the exam that was revoked, and is **`null` for `?exam=all`** — that null
means "all of them", not "none".

### 1.8 Extend trial

`POST /sme/users/:id/trial-extension` — extend a user's free trial by N days, including
reviving trial-expired and churned users. **Full guide (semantics, revival rules, response
shape, recommended notify pairing): [SME_TRIAL_EXTENSION_API.md](./SME_TRIAL_EXTENSION_API.md).**
Audited as `TRIAL_EXTEND`.

**One clock, per-exam length.** The trial is a **person-level** window — one lifetime
clock from signup, spanning every exam, so switching exams can never farm a second
trial — but **how long** that window is belongs to the exam. The extension therefore
applies to the person everywhere, while the end it stacks on is computed from the
policy of the user's own `activeExamId` (`upsc-cse` when unset). **Nothing here is a fixed
number of days by contract**: `upsc-cse` was 14 days until 2026-09-25 and is **3 days** for
signups since (its `trialPolicy`, SME_EXAMS_API §4); `appsc-group-1` is 0.

The response carries two extra fields so the portal can explain a non-obvious number:
`examId` (the exam whose policy produced `previousTrialEndsAt`) and `trialDays` (that
policy's length). A user who is **paying in any exam** is refused with a 400 — the check
reads the person-level mirror, so holding APPSC blocks a UPSC-flavoured trial extension
too. See the trial-extension guide for the exact rule.

### 1.9 Notify a single user
```
POST /sme/users/:id/notify
{ "title": "…", "body": "…", "type": "practice_list", "id": "optional-entity-id",
  "params": { "filterType": "year", "filterValue": "2025" } }
```
Sends an FCM push to the user's active devices and writes it to their in-app feed.
`type` is the deep-link route (`home`, `chat`, `daily_task`, `mains_question`, …;
default `home`). **Response:** `{ "successCount": 1, "failureCount": 0, "deactivatedTokens": 0 }`.

| Field | Required | Description |
|---|---|---|
| title | yes | ≤120 chars |
| body | yes | ≤500 chars |
| type | no | deep-link type, default `home` |
| id | no | entity id forwarded in the payload |
| **params** | no | Extra deep-link arguments for `type`s that need more than an id (e.g. `practice_list`). **Flat string→string map: ≤10 keys, keys 1–40 chars, values ≤256 chars, ≤1 KB serialized.** Values must be strings — a number or boolean is a 400, because the FCM data block is string→string and the app would read back a different type. Delivered as one JSON-encoded `params` data key; app builds that predate it ignore it. |

---

## 2. Transactions (view-only) — `/sme/transactions`, `/sme/orders`, `/sme/webhooks`

Read-only. Razorpay **and** Apple data. Amounts in rupees.

### 2.1 List payments
```
GET /sme/transactions?page=1&limit=20&userId=&paymentStatus=&source=&exam=&startDate=&endDate=&search=&includeTest=
```
| Query | Description |
|---|---|
| paymentStatus | PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED |
| source | RAZORPAY, APPLE, MANUAL (filters by the parent order's source) |
| userId | payments for one user |
| **exam** | Filters on the **parent order's** `exam_id`, exactly as `userId`/`source`/`isTest` already do. Pass a slug, or the literal **`none`** to select payments whose order has **no** exam (the `UNRESOLVED_EXAM` cases, §2.3). **400 on an unknown slug.** |
| startDate / endDate | `YYYY-MM-DD` = IST calendar days (end day fully included); ISO timestamp taken literally |
| search | razorpay payment id / payer email / user email |
| includeTest | `true` to include App Store **sandbox** purchases. Default **false** — see the note under §2.3. (`false` now really parses as false.) |
| includeInternal | `1`/`true` to include internal accounts (default excluded) |

**2026-11 row flags** (every row, always present): `isInternal`, `isComp` (parent order is a
MANUAL staff grant — listed, **not money**; its amount is the notional list price),
`isSandbox` (parent order `is_test`). Money totals must exclude `isComp` and `isSandbox`
rows — or better, read `GET /sme/analytics/summary` → `money` (computed server-side).

**Response 200:**
<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/transactions?exam=appsc-group-1&limit=2` — verbatim, the only matching row:

```json
{
  "data": [
    {
      "id": "6c82ea19-4d53-4071-b198-24fae6d93c75",
      "orderId": "5b71d908-3c42-4f6b-a087-13e9d5c82b64",
      "razorpayPaymentId": "pay_XXXXXXXXXXXXXX",
      "amount": 499,
      "currency": "INR",
      "status": "CAPTURED",
      "method": "card",
      "email": null,
      "contact": null,
      "bank": null,
      "wallet": null,
      "vpa": null,
      "errorCode": null,
      "errorReason": null,
      "capturedAt": "2026-09-16T12:09:03.000Z",
      "createdAt": "2026-09-16T12:09:32.611Z",
      "order": {
        "id": "5b71d908-3c42-4f6b-a087-13e9d5c82b64",
        "userId": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
        "planType": "MONTHLY",
        "paymentSource": "RAZORPAY",
        "examId": "appsc-group-1",
        "isTest": false,
        "user": {
          "id": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
          "email": "aspirant@example.com",
          "name": "A. Sharma",
          "phoneNumber": "+91XXXXXXXXXX"
        }
      }
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 2,
  "hasMore": false
}
```

> **The payer-detail columns are routinely `null`** — `email`, `contact`, `bank`,
> `wallet`, `vpa`, `errorCode`, `errorReason` above are all null on a captured card
> payment. They are populated only when Razorpay's webhook carries them, so the portal's
> payment row must render an absent payer email as "—", not as a loading state.
The nested `order` carries **`examId`** (nullable — see §2.3) and **`isTest`**. The
payment row itself has no exam of its own; a payment's exam is always its order's.

### 2.2 Payment detail
```
GET /sme/transactions/:id
```
Same shape + `notes` (the nested `order.examId` / `order.isTest` included). `404` if not
found. Detail reads are **never** filtered by `includeTest`.

### 2.3 List orders / order detail
```
GET /sme/orders?page=&limit=&userId=&orderStatus=&source=&exam=&startDate=&endDate=&search=&includeTest=
GET /sme/orders/:id
```
`orderStatus` ∈ CREATED, PAID, FAILED, CANCELLED. `exam` takes a slug or the literal
`none` (400 on an unknown slug) — same rule as §2.1, filtering `orders.exam_id` directly.

Each order includes its `payments[]`, `planType`, `paymentSource`, **`examId`**,
**`premiumGrantedAt`**, **`reconcileHint`**, and the six money/provenance fields below.

<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/orders?exam=appsc-group-1` — verbatim, the only matching order:

```json
{
  "data": [
    {
      "id": "5b71d908-3c42-4f6b-a087-13e9d5c82b64",
      "userId": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
      "razorpayOrderId": "order_XXXXXXXXXXXXXX",
      "paymentSource": "RAZORPAY",
      "planType": "MONTHLY",
      "examId": "appsc-group-1",
      "amount": 499,
      "currency": "INR",
      "receipt": "sub_sub_XXXXXXXXXXXXXX",
      "status": "PAID",
      "offerCode": null,
      "listAmount": null,
      "storefrontCurrency": null,
      "chargedAmount": null,
      "isTest": false,
      "premiumGrantedAt": "2026-09-16T12:09:32.623Z",
      "reconcileHint": "OK",
      "createdAt": "2026-09-16T12:09:32.607Z",
      "updatedAt": "2026-09-16T12:09:35.194Z",
      "user": {
        "id": "0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30",
        "email": "aspirant@example.com",
        "name": "A. Sharma",
        "phoneNumber": "+91XXXXXXXXXX"
      },
      "payments": [
        {
          "id": "6c82ea19-4d53-4071-b198-24fae6d93c75",
          "orderId": "5b71d908-3c42-4f6b-a087-13e9d5c82b64",
          "razorpayPaymentId": "pay_XXXXXXXXXXXXXX",
          "amount": 499,
          "currency": "INR",
          "status": "CAPTURED",
          "method": "card",
          "email": null,
          "contact": null,
          "bank": null,
          "wallet": null,
          "vpa": null,
          "errorCode": null,
          "errorReason": null,
          "capturedAt": "2026-09-16T12:09:03.000Z",
          "createdAt": "2026-09-16T12:09:32.611Z"
        }
      ]
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 20,
  "hasMore": false
}
```

> **`appleTransactionId` is absent, not null, on a Razorpay order** — and
> `razorpayOrderId` is absent on an Apple one. Read whichever matches
> `paymentSource`; do not assume both keys exist on every row.
>
> The Apple-only money fields (`storefrontCurrency`, `chargedAmount`) and `offerCode` /
> `listAmount` are all `null` above because this is a plain full-price Razorpay
> purchase. **Staging carries no Apple orders at all** (`GET /sme/orders?source=APPLE`
> returns `{"data":[],"total":0,"page":1,"limit":20,"hasMore":false}` — captured
> 2026-09-21), so a populated Apple row could not be captured; see
> [`SME_APPLE_IAP_API.md`](./SME_APPLE_IAP_API.md).

| Field | Meaning |
|---|---|
| `examId` | Which exam the money bought. **Nullable on purpose** — Apple's verify/webhook payloads carry no exam, so a product added in App Store Connect but never seeded into `exam_plans` produces a real charge with no exam rather than a guessed one. |
| `offerCode` | The campaign the purchase was made under, or `null`. This is the **only** attribution rule the campaign funnel uses (see `SME_OFFERS_API.md`). |
| `listAmount` | ₹, the **undiscounted** price at the moment of purchase, or `null`. Without it a discounted order is indistinguishable from a price change after the fact. |
| `storefrontCurrency` | Apple's **true** storefront currency for this buyer (`"USD"`, `"AED"`, …). `null` for every Razorpay/MANUAL order and for Apple transactions where Apple supplied no price (pre-2023 JWS). |
| `chargedAmount` | What Apple **actually charged**, in that storefront's currency, as a major unit. **Independent of `amount`/`currency` above, which stay INR-only:** for a non-INR Apple order, `amount` is the INR *list* price, not a real charge. Never add the two together, and never sum `amount` across storefronts and call it revenue. |
| `isTest` | App Store **sandbox** purchase — a real order row, not real money. |

**Reconcile workflow — `reconcileHint`.** Computed server-side so the portal never
re-derives it, and split into two buckets because they need **opposite** fixes:

| `reconcileHint` | Means | Fix |
|---|---|---|
| `OK` | Premium was granted, **or** the order never became money (not `PAID`). | Nothing to do. |
| `GRANT_PENDING` | `status=PAID`, **exam known**, `premiumGrantedAt=null`. A webhook that never landed, or a failed verify. | Re-run the grant via §1.6 **with the order's own `examId`** — never the default, never the user's `activeExamId`. |
| `UNRESOLVED_EXAM` | `status=PAID`, `premiumGrantedAt=null`, **`examId` is null**. An Apple product with no `exam_plans` row: we took real money and do not know what was bought. Premium is **deliberately** not granted. | **Seed the `exam_plans` row first** (map the `appleProductId` to its exam + plan), then grant with the right `examId`. **Never grant one of these blind** — granting a guessed exam for real money is worse than granting nothing. |

Filter that last bucket directly with **`?exam=none`**. Sandbox orders are *not*
special-cased by the hint: a test purchase that failed to grant is still a broken grant
path, and it is already labelled `isTest`.

> ⚠️ **Changed 2026-09-21.** The old rule — "`PAID` + `premiumGrantedAt=null` → grant it
> via §1.6" — is no longer safe on its own. It is correct for `GRANT_PENDING` and wrong
> for `UNRESOLVED_EXAM`. Branch on `reconcileHint`, not on the two raw columns.

> **Sandbox orders are hidden by default.** iOS sandbox purchases reach the production
> backend (Apple has no separate endpoint), so every payment test writes a genuine PAID
> order at the real price — one evening of testing produced ₹59,000 of phantom revenue.
> Those orders carry `isTest: true` and are excluded from `/sme/orders` **and**
> `/sme/transactions` unless you pass `includeTest=true`. Since the portal computes
> revenue by summing these lists, the default is what keeps the headline honest.
> Detail reads (`/sme/orders/:id`, `/sme/transactions/:id`) and a user's `recentOrders`
> are never filtered — they show `isTest` instead.

> **Apple amounts: `amount` is INR-only, `chargedAmount` is the truth.** An App Store
> offer-code redemption is billed below list price; for an **INR** storefront the order
> records the charged amount in `amount`, the list price in `listAmount`, and the
> campaign in `offerCode`. For a **non-INR** storefront `amount` falls back to the
> configured INR list price with `listAmount: null` — it is *not* a real charge — while
> `storefrontCurrency` + `chargedAmount` carry what Apple actually billed. Both of those
> are `null` only when Apple supplied no price at all (pre-2023 transaction). `amount`
> keeps its INR-paise meaning on purpose: every existing revenue sum already reads it
> that way, so mixing currencies into it would corrupt history rather than fix it.

### 2.4 Webhook / provider events
```
GET /sme/webhooks?page=&limit=&source=&startDate=&endDate=&search=&kind=&includeInternal=
```

**2026-11 (B1.4):** `kind=money|downtime|all` (default `all`) — `money` hides the Razorpay
`payment.downtime.*` gateway notices (~99.5% of rows), `downtime` shows only them. New row
fields: `isDowntime`, `isSandbox` (Apple: decoded JWS environment), `isComp` (always
false), `isInternal`. Events are now linked to `userId`/`orderId` at ingest where the
payload allows (older rows: `scripts/backfill-payment-event-links.ts`, owner runs it on
prod), so `examId` and the per-user webhook count stop being empty for everyone.
Provider event ledger from `payment_events` — **Razorpay webhooks + Apple App Store
notifications**. Filter `source=RAZORPAY|APPLE` (omit for both; `MANUAL` = empty). Keeps
the original keys (`eventId`, `eventType`, `source`, `payload`, `processedAt`) and adds
`provider`, `kind`, `result`, `error`, `userId`, `orderId`, **`examId`**, `receivedAt`.

`examId` is read off **the order the event was matched to** — `payment_events` carries no
exam of its own, it is the raw provider envelope. It is `null` when the event matched no
order (an unknown Apple product, a webhook for a foreign account) or when the order it
matched has no resolved exam. That null is the honest answer, not a gap.

> **`?exam=` does not filter this list.** It is accepted by the shared query DTO but
> deliberately ignored here: the ledger is the raw envelope, and hiding events by an exam
> they were never stamped with would hide exactly the unmatched ones a reconcile pass is
> looking for. Filter client-side on the returned `examId` if you need to.

> ⚠️ This endpoint used to read a legacy table that was never written, so it returned an
> empty list. It now returns real events — **additive/non-breaking** (same keys, plus new
> ones). Full Apple detail (notification types, lifecycle, field reference) is in
> [SME_APPLE_IAP_API.md](./SME_APPLE_IAP_API.md).

---

### 2.5 Payment attempts — `GET /sme/payment-attempts` (2026-11, B1.2)
```
GET /sme/payment-attempts?page=&limit=&userId=&exam=&kind=&status=&from=&to=&search=&includeInternal=
```
One row per attempt to pay, newest first: every order (Apple, Razorpay one-time, MANUAL
comp) **plus every Razorpay UPI AutoPay mandate** from `subscriptions` — which creates no
order until it is charged, so before this route a customer who tried and was never
charged was invisible. A mandate's charge orders are folded into the mandate row.

| Query | |
|---|---|
| kind | `APPLE`, `RAZORPAY_ONE_TIME`, `UPI_AUTOPAY_MANDATE`, `COMP` |
| status | verbatim: orders `CREATED/PAID/FAILED/CANCELLED`, mandates `PENDING/ACTIVE/PAST_DUE/PAUSED/CANCELLED/EXPIRED` |
| exam | slug or `none`; unknown → 400 |
| from / to | IST calendar days |
| search | email / name / phone, or the FULL external id |

Row: `{ id, userId, kind, status, amountPaise|null, currency, examId|null, planType|null,
createdAt, lastEventAt, ref, source: 'orders'|'subscriptions', isInternal, isSandbox,
everCharged, user: { email, name, phoneNumber } }` — `amountPaise` is in **paise** here
(unlike the rupee amounts above); `ref` is the external id cut to 8 chars + `…`. Page:
`{ data, total, page, limit, hasMore, includeInternal, note }`. Full reference and example:
[WHAT_CHANGED_2026-11.md §B1.2](./WHAT_CHANGED_2026-11.md).

---

## 3. Blog Management — `/sme/blogs`

Works **in tandem with reels** (1 blog per reel). **Build the blog editor into the reel
management screen, not a separate screen** — see
`../superpowers/specs/2026-06-16-sme-blog-reels-design.md`. Markdown format = the
content/image/quiz syntax from `CONTENT_DOC_SME_API.md` (quiz blocks are stripped for blogs).

| Method | Path | Body | Notes |
|---|---|---|---|
| GET | `/sme/blogs?page=&limit=&hasBlog=&search=` | — | reels + `hasBlog` + blog summary |
| GET | `/sme/blogs/:reelId` | — | full blog (title, rawMarkdown, sections) |
| POST | `/sme/blogs` | `{ reelId, title, rawMarkdown }` | create; parses sections |
| PATCH | `/sme/blogs/:reelId` | `{ title?, rawMarkdown? }` | re-parses if rawMarkdown sent |
| DELETE | `/sme/blogs/:reelId` | — | deletes the blog; reel untouched |

**List response 200** `data[]`:
```json
{ "reelId": "…", "title": "…", "subject": "Polity", "status": "READY",
  "date": "…", "hasBlog": true,
  "blog": { "id": "…", "title": "…", "createdAt": "…", "updatedAt": "…" } }
```
**Create/Get response:** the `ReelBlog` row incl. `sections` (array of
`{ order, type: "content", markdown }` / `{ order, type: "image", url, alt }`).
`404` on get/update/delete if the reel has no blog.

---

## 4. Notifications — `/sme/notifications`

> **Full guide: [`SME_NOTIFICATIONS_API.md`](./SME_NOTIFICATIONS_API.md)** — covers per-user
> vs broadcast vs segment, the deep-link `type` contract, how premium/free is decided
> (topic subscription vs segment query), and all delivery caveats. The summary below is
> the quick reference.

Single-user notify lives at §1.9. These are the audience sends.

### 4.1 Broadcast to all — or to one exam mode
```
POST /sme/notifications/broadcast
{ "title": "…", "body": "…", "type": "home", "id": "optional",
  "exam": "appsc-group-1" }
```
Sends to the `all_users` FCM topic, or — with **`exam`** — to that exam's topic instead.
**Response** (both paths):
<!-- shape verified against code @c0a8fe5 — NOT captured on staging, see note -->
```json
{ "messageId": "…", "topic": "exam_appsc-group-1" }
```

> ⚠️ **This is the one success body in this doc that is still shape-from-code.** A
> successful send cannot be captured on staging: staging's Firebase service account
> cannot mint an access token, so the send fails at the FCM call with a 500 mentioning
> `iam.serviceAccounts.getAccessToken`. **The success shape was verified on the
> production credential path on 2026-09-17 for the all-users case** (`topic:
> "all_users"`); the exam-topic branch differs only in the `topic` string it returns.

`topic` is `all_users` when `exam` is omitted, so the portal never has to branch on its
own request to state the audience. Topic names keep the slug verbatim, hyphens included.

**`exam` is validated, and the validation was captured** — it runs *before* the FCM call,
so it is reachable on staging.
<!-- captured from staging 2026-09-21, backend f6329e6 -->
`POST /sme/notifications/broadcast` with `{"title":"x","body":"y","type":"home","exam":"appsc-grp-1"}`
→ **400**, nothing sent:

```json
{
  "success": false,
  "message": "Unknown exam \"appsc-grp-1\". Create it via POST /sme/exams first.",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:38:25.811Z",
  "path": "/api/v1/sme/notifications/broadcast",
  "method": "POST"
}
```

That guard exists because a topic send to a nonexistent topic returns a healthy
`messageId` and reaches nobody. `exam: "upsc-cse"` also reaches everyone who has never
used the exam picker.

⚠️ **Coverage caveat:** a device joins `exam_<slug>` on its next dashboard sync, so an
exam broadcast reaches a growing subset rather than everyone on day one. No app release
is needed. Full detail — including feed persistence — in
[`SME_NOTIFICATIONS_API.md` §2](./SME_NOTIFICATIONS_API.md).

### 4.2 Segment send
```
POST /sme/notifications/segment
{ "segment": "premium", "title": "…", "body": "…", "type": "home", "id": "optional",
  "exam": "appsc-group-1" }
```
`segment` ∈ `premium` (holds a live entitlement — paid **or** staff comp) / `trial` (active
free trial) / `free` (everyone else). **Size it first, without sending:**
`GET /sme/notifications/segment/count?segment=&exam=` → `{ matchedUsers, segment, exam,
examDefaulted }`, the same audience method as the send (lane B6.4).
**2026-11 (B3.2 + owner rule):** the cohorts are the §1.1a access axis collapsed —
`premium` = holds a **live** entitlement (`paid` ∪ `comped`, incl. the legacy mapping),
`trial` = access `trial` (each exam's own trial length — it was a fixed 14 days for
everyone: prod "Cohort: Trial" reached 1,609 vs 274 on trial), `free` = everyone else.
Scope = `exam` when sent (its entitlement rows + its exam mode), otherwise every user and
rows of any exam. Also fixed: `free` dropped every ACTIVE user past their trial (a NULL
`trial_ends_at` made NOT(…) NULL) — they were in no cohort at all.
Resolves matching users and fans out a per-user push. **Response:**
<!-- shape verified against code @c0a8fe5 — NOT captured on staging, see note -->
```json
{ "segment": "premium", "exam": "appsc-group-1", "matchedUsers": 412,
  "successCount": 380, "failureCount": 32 }
```

> ⚠️ **Shape-from-code, for the same reason as §4.1** — staging's Firebase credential
> cannot mint an access token, so a real fan-out 500s at the FCM call
> (`iam.serviceAccounts.getAccessToken`). Verified on the production credential path on
> 2026-09-17 for the all-users case.

The **exam validation** *was* captured — it runs before any resolution or send:
<!-- captured from staging 2026-09-21, backend f6329e6 -->
`POST /sme/notifications/segment` with
`{"segment":"premium","title":"x","body":"y","type":"home","exam":"appsc-grp-1"}` →
**400**, nothing resolved, nothing sent:

```json
{
  "success": false,
  "message": "Unknown exam \"appsc-grp-1\". Create it via POST /sme/exams first.",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:38:26.106Z",
  "path": "/api/v1/sme/notifications/segment",
  "method": "POST"
}
```

`exam` is echoed back (`null` on the un-scoped call) so the portal can label the send
with the audience the server actually used.

**`exam` (optional)** narrows the cohort to one exam mode, so a state-exam announcement
does not reach UPSC aspirants. Omit it to reach the whole cohort — exactly the previous
behaviour. Matching is on the user's last switcher choice, and **`exam: "upsc-cse"` also
matches every user who has never touched the switcher** (an unset preference means UPSC),
which is the same rule `GET /sme/users?exam=` uses. The cohorts are NOT read from the
`user_auth` mirror: they come from `user_exam_entitlements` + each exam's trial clock, and
with `exam` sent `premium` = a live entitlement in **that** exam (without `exam`: in any
exam).

> ✅ **Corrected 2026-09-21 — this parameter is now validated** like every other
> `?exam=` in this doc: an unknown slug is a **400**, not a send to nobody reported as
> `matchedUsers: 0`.
>
> 🔴 **And it used to target the wrong people.** Before this release, `segment: "trial"`
> or `"premium"` combined with `exam: "upsc-cse"` dropped the cohort predicate — a
> "trial" push scoped to UPSC went to the **entire free base**. Fixed; `free` and all
> un-scoped sends were never affected. Detail in
> [`SME_NOTIFICATIONS_API.md` §3.1](./SME_NOTIFICATIONS_API.md).

---

## 5. Existing content handoff — ✅ ALREADY IMPLEMENTED & LIVE (do NOT rebuild)

> **For the SME portal agent:** everything in this section is **already shipped and in
> production**. It is listed here only so you have the complete picture. **Do not
> re-implement or re-scope it** — the SME portal already integrates these. The NEW work
> in this doc is §1–§4 and §6 (`/sme/*`). The only NEW content-adjacent work is **blog
> management (§3)**, which builds on the existing reels endpoints below.

These are `@Public()` with **no** `x-api-key` (unlike the `/sme/*` routes above). Full guides:

- **`CMS_ADMIN_API.md`** — `/cms/pyq`, `/cms/mains`, `/cms/psychometric` (PYQ / Mains /
  Psychometric CRUD + bulk, paginated lists). **DONE.**
- **`CONTENT_DOC_SME_API.md`** — `/content-doc-admin` (study documents from raw markdown
  with embedded quiz blocks → parsed sections). **DONE.**
- **Reels / video** — `GET /reels` (list, filters), `POST /videos/upload-url` (Mux upload),
  `PUT /reels/bulk` (metadata after processing), `DELETE /reels/:videoId`. **DONE.** Blogs
  for these reels are the only NEW piece — managed via §3.
- A key-gated `/sme/content/*` ingest surface is landing in this release — see
  [SME_CONTENT_INGEST_API.md](./SME_CONTENT_INGEST_API.md), and
  [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md) §10 for the migration map.

**Blog render note (verified from client code):** blog `sections` produced by §3 render
correctly on both mobile and web with **zero** extra formatting — the apps consume the
stored `sections` verbatim (`{order, type:'content', markdown}` / `{order, type:'image',
url, alt}`, lowercase discriminators) via `GET /reels/:reelId/blog`. The SME blog
endpoints reuse the same parser as the existing live blog admin, so output is identical.

---

## 6. Subject Filter Config — `/sme/filter-config`

Per-subject display name / icon / visibility / sort order for the app's PYQ Prelims and
Mains filters, plus flipping a mains subject between Optional and GS. **Full guide
(endpoints, icon-key list, override semantics, cache propagation):
[SME_FILTER_CONFIG_API.md](./SME_FILTER_CONFIG_API.md).** Audited as
`FILTER_CONFIG_UPDATE` / `MAINS_OPTIONAL_FLIP`.

---

## Common errors

| Status | Cause | Fix |
|---|---|---|
| 401 | Missing/invalid `x-api-key` | Send the `x-api-key` header = `API_KEY_SECRET` |
| 400 | Validation (bad enum, missing required field) | Check the body against the tables above |
| 400 | Unknown `exam` slug on any `?exam=` parameter | Source the slug from `GET /sme/exams`. Every new exam parameter 400s rather than returning an empty page — an empty management screen reads as "this exam has nothing" |
| 400 | `DELETE /sme/users/:id/premium` with no `?exam=` | **BREAKING, §1.7** — pass a slug, or `?exam=all` |
| 404 | Unknown user / payment / order / blog | Verify the id |

## Audit trail

Every privileged/destructive SME action (update, deactivate, hard-delete, premium
grant/revoke, trial extend, notify, blog create/update/delete, filter-config update,
mains optional flip) is recorded in `sme_audit_log` (action, target user, before/after
snapshot, optional reference, timestamp).
