# SME — Trial Extension API Guide

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

**Semantics:**
- Adds `days` on top of `max(now, current trial end)` — i.e. it always extends
  forward from "whichever is later: right now, or the trial end already in
  effect." It never rewinds an end date, and back-to-back calls **stack**: there
  is no lifetime cap, every call is independently audited.
- Works for a user who is: **currently in-trial** (just pushes the end further
  out), **trial-expired** (never paid, window closed), or **churned**
  (previously paid, now lapsed). For the latter two, the user is **revived** —
  `status` flips back to `ACTIVE` — and they get **full premium access** until
  the new trial end, exactly like being freshly inside the 14-day window.
- Sends **no push notification** — see "Recommended portal UX" below.
- **400** if the user currently has an **active paid subscription**
  (`status=SUBSCRIBED` and not expired) — use premium grant
  (`SME_PORTAL_API.md` §1.6) instead.
- **400** if the user is `SUSPENDED`, `LOCKED`, `ONBOARDING`, or `INACTIVE`
  (non-standard lifecycle states — reactivate/onboard them first).
- **404** if the user doesn't exist.

**Response 200:**
```json
{
  "success": true,
  "trialEndsAt": "2026-08-15T10:30:00.000Z",
  "trialDaysLeft": 26,
  "statusChanged": true,
  "previousTrialEndsAt": "2026-07-06T10:30:00.000Z"
}
```
`statusChanged` is `true` when the call revived a lapsed/churned user (`status`
went to `ACTIVE`); `false` when it just extended an already-active trial.
`previousTrialEndsAt` is the trial end that was in effect right before this call
(useful for an undo/audit UI).

## 2. Recommended portal UX — pair with "Notify user"

Pair the "Extend trial" action with a **"Notify user"** button that calls
`POST /sme/users/:id/notify` (`SME_PORTAL_API.md` §1.9) right after a successful
extension. The backend deliberately does **not** push a notification on extension —
the portal owns the messaging (wording, timing, whether to notify at all), so wire
the two actions together in the UI rather than assuming the user was told.

## 3. Where trial state is visible

- SME user list/detail (`SME_PORTAL_API.md` §1.1/§1.2) expose `trialEndsAt`,
  `trialDaysLeft` (and `trialExtended` on detail).
- The apps read `isTrial` / `trialEndsAt` / `trialDaysLeft` (the EFFECTIVE end)
  from `GET /payments/entitlement` and `GET /user/profile/me`.
- The user's Wylto contact re-derives its funnel status (e.g. back to
  `FREE_TRIAL_ACTIVE`) immediately after the extension.

---

## Common errors

| Status | Cause | Fix |
|---|---|---|
| 401 | Missing/invalid `x-api-key` | Send the `x-api-key` header = `API_KEY_SECRET` |
| 400 | Active paid subscription, or non-standard lifecycle state, or days out of 1–365 | See semantics above |
| 404 | Unknown user id | Verify the id |

Every call is recorded in `sme_audit_log` (`TRIAL_EXTEND`) with before/after payloads.
