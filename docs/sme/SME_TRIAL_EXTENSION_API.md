# SME — Trial Extension API Guide

> Changed 2026-09-21 — exam dimension on users, orders, offers, banners; two BREAKING calls (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

Reference for the **SME portal** to extend a user's free trial by N days — including
reviving trial-expired and churned users. Same server-to-server model as the rest of the
SME surface: we expose the endpoint, the SME team builds the UI.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `BASE_URL` = `https://app.stanzasoft.ai`)
**Authentication:** `x-api-key: <API_KEY_SECRET>` header on **every** request.
**Swagger:** `/api/docs` (group **SME**) documents this live with the `x-api-key` control.

---

## 1. Extend trial
```
POST /sme/users/:id/trial-extension
{ "days": 14, "reason": "Escalated support ticket #4821 — gave 2 extra weeks" }
```
| Field | Required | Description |
|---|---|---|
| days | yes | integer, 1–365 |
| reason | no | free text, ≤200 chars — **not** shown to the user, stored only on the `sme_audit_log` row (`TRIAL_EXTEND`) |

> ### One clock, per-exam length
>
> The trial is a **person-level** window: one lifetime clock from signup, spanning every
> exam. That is deliberate — a per-exam trial would hand a user N × the free days just
> for tapping the exam switcher. **How long** that window is, though, belongs to the
> exam (`exams.trial_policy`), resolved against the user's own `createdAt` so shortening
> an exam's trial never reaches back and ends a window someone is already inside.
>
> So an extension is a grant to the **person** and applies in every exam, while the end
> it **stacks on** is computed from the policy of the user's `activeExamId` (`upsc-cse`
> when they have never picked one). **Nothing here is 14 days by contract** — 14 is
> merely `upsc-cse`'s current length. Do not hard-code it in the portal; read `trialDays`
> off the response (below) or `trialExamId` off the user row.

**Semantics:**
- Adds `days` on top of `max(now, current trial end)` — i.e. it always extends
  forward from "whichever is later: right now, or the trial end already in
  effect." It never rewinds an end date, and back-to-back calls **stack**: there
  is no lifetime cap, every call is independently audited.
- Works for a user who is: **currently in-trial** (just pushes the end further
  out), **trial-expired** (never paid, window closed), or **churned**
  (previously paid, now lapsed). For the latter two, the user is **revived** —
  `status` flips back to `ACTIVE` — and they get **full premium access** until
  the new trial end, exactly like being freshly inside their exam's trial window.

**Re-activation, and why it is needed.** `PremiumExpiryGuard` writes
`user_auth.status = 'UNSUBSCRIBED'` on the user's next request as soon as an account's
trial window closes — which is also what happens when an SME **ends a trial early** by
moving `trial_ends_at` into the past. Access gates (`hasExamAccess`) admit the trial
branch only for `ACTIVE`/`SUBSCRIBED`, so pushing `trial_ends_at` back into the future
is not enough on its own: without a matching status flip the account would stay
permanently non-trial until someone hand-wrote the column. This endpoint therefore also
sets `status = 'ACTIVE'`, when two things hold: the current status is exactly
`UNSUBSCRIBED`, and the new effective trial end is in the **future** (an extension that
still lands in the past grants nothing). A **paying** customer is never touched — a
`SUBSCRIBED` user keeps that status, because demoting them to `ACTIVE` would break
`hasPaidEntitlement`, the buy flow, Wylto and every SME report at once, and a stale
`premium_expires_at` mirror is not evidence they have lapsed. A **churned** customer is
deliberately revivable: a goodwill extension for someone who used to pay is the main
thing this endpoint is for. `SUSPENDED`/`LOCKED`/`ONBOARDING`/`INACTIVE` are rejected
with a 400 before any of this runs.
- Sends **no push notification** — see "Recommended portal UX" below.
- **400** if the user currently has an **active paid subscription**
  (`status=SUBSCRIBED` and not expired) — use premium grant
  (`SME_PORTAL_API.md` §1.6) instead.

> ### ⚠️ "Paid in ANOTHER exam" is also refused
>
> That 400 is evaluated against the **person-level mirror** (`user_auth.status` +
> `premiumExpiresAt`), which flips to `SUBSCRIBED` the moment the user buys **any** exam.
> So a user with a live ₹499 APPSC subscription **cannot be given a UPSC trial extension**
> — the call returns:
>
> ```
> User has an active paid subscription — use POST /sme/users/:id/premium to extend premium instead.
> ```
>
> This is the current behaviour, not an oversight to route around: the trial is
> person-level, and this person's trial clock is not what is gating them. If the intent
> is to give them free time in an exam they have *not* bought, use a **manual grant**
> for that exam (`POST /sme/users/:id/premium` with `examId`) instead. Check the user's
> `entitlements[]` (`SME_PORTAL_API.md` §1.1) before offering the "Extend trial" button
> — if any row is `LIVE`, the button will 400.
- **400** if the user is `SUSPENDED`, `LOCKED`, `ONBOARDING`, or `INACTIVE`
  (non-standard lifecycle states — reactivate/onboard them first).
- **404** if the user doesn't exist.

<!-- captured from staging 2026-09-21, backend f6329e6 -->
**Both refusals above were captured on staging on 2026-09-21** (backend `f6329e6`).
`POST /sme/users/0d2f8b41-…/trial-extension` with `{"days":7}`, against a user holding a
live APPSC subscription → **400**, and the `message` is exactly the sentence quoted in
the box above:

```json
{
  "success": false,
  "message": "User has an active paid subscription — use POST /sme/users/:id/premium to extend premium instead.",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-09-21T14:57:49.275Z",
  "path": "/api/v1/sme/users/0d2f8b41-9a3c-4f2e-8c71-2b6d5a1e7f30/trial-extension",
  "method": "POST"
}
```

The same call against an id that does not exist → **404**, `message: "User not found"`
(note: a plain string, not the id-echoing message some other routes use):

```json
{
  "success": false,
  "message": "User not found",
  "error": "Not Found",
  "statusCode": 404,
  "timestamp": "2026-09-21T14:57:49.361Z",
  "path": "/api/v1/sme/users/00000000-0000-4000-8000-000000000000/trial-extension",
  "method": "POST"
}
```

**Response 200** — shape from code; **not captured**, because a successful extension
mutates a real user's trial clock and staging's users are shared fixtures:
<!-- shape verified against code @4f6622e — success path not run; both refusals ARE captured above -->
```json
{
  "success": true,
  "trialEndsAt": "2026-08-15T10:30:00.000Z",
  "trialDaysLeft": 26,
  "statusChanged": true,
  "previousTrialEndsAt": "2026-07-06T10:30:00.000Z",
  "examId": "appsc-group-1",
  "trialDays": 7
}
```
`statusChanged` is `true` when the call revived a lapsed/churned user (`status`
went to `ACTIVE`); `false` when it just extended an already-active trial, or when
the user was left on `SUBSCRIBED`.
`previousTrialEndsAt` is the trial end that was in effect right before this call
(useful for an undo/audit UI).

| Field | Meaning |
|---|---|
| `examId` | The exam whose trial policy produced `previousTrialEndsAt` — the user's `activeExamId`, or `upsc-cse` when they have never picked one. **Not** a scope: the extension itself spans every exam. |
| `trialDays` | That policy's length in days. It is what `previousTrialEndsAt` was computed from, and it is the one thing an SME cannot otherwise see when the answer is not 14. |

> Render these two together, e.g. *"stacked on the **7-day appsc-group-1** window."*
> Without them, a portal that assumes 14 shows a `previousTrialEndsAt` that looks wrong
> by a week and generates a support ticket about the support tool.

## 2. Recommended portal UX — pair with "Notify user"

Pair the "Extend trial" action with a **"Notify user"** button that calls
`POST /sme/users/:id/notify` (`SME_PORTAL_API.md` §1.9) right after a successful
extension. The backend deliberately does **not** push a notification on extension —
the portal owns the messaging (wording, timing, whether to notify at all), so wire
the two actions together in the UI rather than assuming the user was told.

## 3. Where trial state is visible

- SME user list/detail (`SME_PORTAL_API.md` §1.1/§1.2) expose `trialEndsAt`,
  `trialDaysLeft` (and `trialExtended` on detail), plus **`trialExamId`** — the exam
  whose policy produced that window, on **every list row**. Show it next to the date:
  it is the difference between "this looks wrong" and "this exam grants 7 days."
- `entitlements[].isTrial` on the same rows says, **per exam**, whether the
  person-level trial is what is currently carrying the user in that exam (only ever
  true once that exam's own entitlement row has stopped granting anything).
- The support snapshot (`SME_ACTIVITY_TRAIL_API.md`) and the support-code lookup
  resolve the same per-exam length, so the header, the list row and the snapshot now
  agree for users whose exam is not `upsc-cse`.
- The apps read `isTrial` / `trialEndsAt` / `trialDaysLeft` (the EFFECTIVE end)
  from `GET /payments/entitlement` and `GET /user/profile/me`.
- The user's Wylto contact re-derives its funnel status (e.g. back to
  `FREE_TRIAL_ACTIVE`) immediately after the extension.

---

## Common errors

| Status | Cause | Fix |
|---|---|---|
| 401 | Missing/invalid `x-api-key` | Send the `x-api-key` header = `API_KEY_SECRET` |
| 400 | Active paid subscription **in any exam**, or non-standard lifecycle state, or days out of 1–365 | See semantics above |
| 404 | Unknown user id | Verify the id |

Every call is recorded in `sme_audit_log` (`TRIAL_EXTEND`) with before/after payloads.
