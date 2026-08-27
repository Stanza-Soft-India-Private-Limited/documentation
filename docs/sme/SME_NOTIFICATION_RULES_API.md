# SME — Notification Rules API Guide

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Auth:** `x-api-key: <API_KEY_SECRET>` on every request.
**No response envelope** — branch on HTTP status, never on a `success` field.

> ⚠️ **Invoke the `frontend-design` skill before building any screen from this doc.**

---

## 0. The mental model (read this first)

A **rule** is one notification the portal owns end to end: its words, its send time, its
thresholds, its audience, its deep link, and whether it is on at all.

What the rule does **not** own is *who qualifies*. That predicate is code, selected by
`triggerKey`. So:

| Change | Needs a deploy? |
|---|---|
| Reword a notification | **No** |
| Move it from 20:30 to 19:00 | **No** |
| Add a second reminder at T-7 alongside T-3 | **No** — create another rule on the same trigger |
| Only send it to trial users on Android | **No** |
| Only send it to APPSC users | **No** |
| Turn it off right now | **No** |
| Invent a *new kind* of condition we have never computed | **Yes** |

The last row is the whole boundary. Everything else is a row edit.

### One trigger, many rules

`triggerKey` is not unique. A T-7 / T-3 / T-1 ladder is **three rules on
`subscription_ending`**, each with its own `config.daysBefore` and its own copy. An
APPSC-specific variant is another rule with `examId` set.

### Nothing sends until you say so

Three independent brakes, and all three must be released:

1. **`isEnabled`** — every rule ships `false`.
2. **`rolloutPercent`** — defaults to `0`. **Enabling a rule is not the same as sending it.**
   A rule that is enabled at 0% reaches nobody. Ramp 5 → 25 → 100.
3. **The global kill switch** (`GET/PATCH .../settings`) — freezes everything at once.

Meanwhile a **disabled rule is still evaluated** on its schedule, in shadow: it resolves its
real audience, applies the real suppression chain, records the counts, and sends nothing. So
by the time you enable something you already know what it would have done.

### The two rules that ship ON

`payment_success` and `trial_ending` are seeded **enabled at 100%**, because they already fire
in production today. Moving them into the engine is a consolidation, not a launch. Every other
rule ships off.

---

## 1. The trigger registry

```
GET /sme/notification-rules/triggers
```

**Build the rule form from this response.** It carries each trigger's config schema, the
template variables its copy may use, and its defaults — so a trigger added later appears in
the portal with no portal change.

```jsonc
[
  {
    "key": "subscription_ending",
    "kind": "SWEPT",                  // SWEPT = runs on a clock; EVENT = fires on a domain event
    "category": "PAYMENT",
    "description": "Premium expires in N days. Create one rule per step of a ladder.",
    "configSchema": [
      { "key": "daysBefore", "label": "Days before", "type": "int",
        "min": 0, "max": 30, "default": 3,
        "help": "Fires on the day premium is exactly this many days from expiring." }
    ],
    "defaultConfig":    { "daysBefore": 3 },
    "defaultCopy":      { "title": "Your Premium ends in {{daysLeft}} days", "body": "..." },
    "defaultDeepLink":  { "type": "premium" },
    "defaultSendAtIst": "10:00",
    "vars": ["name", "daysLeft", "expiresOn"],
    "transactional": false,
    "available": true,
    "unavailableReason": null
  }
]
```

- **`kind: "EVENT"`** — fires on something happening (a payment, a signup), not on a clock.
  `sendAtIst` does not apply and is rejected on write.
- **`transactional: true`** — bypasses quiet hours, the daily cap and the category opt-out.
  Reserved for messages the user must receive (their payment went through). Not settable by
  the portal; it comes from the trigger.
- **`available: false`** — the trigger exists but cannot run yet. `unavailableReason` says
  why, and `POST /enable` returns 400. Today this is only `pib_today` (see §9).

### The 13 triggers

| Trigger | Kind | Category | Fires when |
|---|---|---|---|
| `subscription_ending` | SWEPT | PAYMENT | Premium expires in exactly `daysBefore` days |
| `trial_ending` | SWEPT | PAYMENT | Trial ends in exactly `daysBefore` days |
| `payment_success` | EVENT | PAYMENT | A payment grants premium |
| `subscription_ended` | EVENT | PAYMENT | Premium lapses and the account drops to free |
| `trial_started` | EVENT | PAYMENT | First signup |
| `streak_unfilled` | SWEPT | STREAK | Today's streak is incomplete (or untouched) |
| `pyq_reminder` | SWEPT | STREAK | No PYQ attempted in `inactiveDays` days |
| `streak_completed` | EVENT | STREAK | Today's streak crosses the completion threshold |
| `continue_learning` | SWEPT | PROGRESS | A part-read document has sat for `inactiveDays` days |
| `weekly_progress` | SWEPT | PROGRESS | Weekly summary |
| `reels_today` | SWEPT | CONTENT | New reels published today — **silent if none** |
| `pib_today` | SWEPT | CONTENT | ⚠️ Not implemented — see §9 |
| `custom` | SWEPT | CONTENT | **Your own message to a cohort** — feature announcements, one-offs |

**`custom` is the escape hatch.** It contributes no predicate of its own: the audience is
whatever `audience` narrows to. Use it for "new feature released" and for any campaign we have
not built a trigger for. That is what keeps a one-off announcement from needing us.

---

## 2. Copy and template variables

Copy is language-keyed. Only `en` is populated today; adding another language later is a data
edit, not a migration.

```json
{ "copy": { "title": "Don't break your {{streak}}-day streak", "body": "..." } }
```

`{{placeholders}}` are filled from the trigger's `vars`. **An unknown or missing variable
renders as an empty string, never as the literal `{{name}}`** — a user must never see template
syntax in their notification shade. Whitespace is collapsed afterwards, so
`"Hi {{name}}, ..."` degrades to `"Hi, ..."` rather than `"Hi , ..."`.

Limits: title ≤ 120 chars, body ≤ 500. Longer values are truncated at send.

---

## 3. Deep links

```json
{ "deepLink": { "type": "premium", "id": null, "params": { "subject": "Polity" } } }
```

`type` must be a value the app's `DeepLinkMapper` knows — the full table is in
[SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) §4.2. An unknown `type` routes Home
rather than failing, so a typo is silent: check the table.

`params` obeys the same caps as the ad-hoc send API — ≤10 keys, key ≤40 chars, value ≤256,
≤1 KB serialized. The engine adds one key of its own, `rid` (the rule id), for tap attribution.

---

## 4. Audience narrowing

Applies to **every** rule, on top of whatever its trigger already selected.

```jsonc
{
  "audience": {
    "tiers": ["trial", "free"],        // premium | trial | free
    "platforms": ["android"],          // android | ios | web (any active device matches)
    "activeWithinDays": 14,            // only users seen in the last N days
    "inactiveForDays": 3               // only users NOT seen for at least N days
  }
}
```

Omit a key to not filter on it. `{}` means "no narrowing".

⚠️ **Tier here is the accurate definition** — `premium` = paid and current, `trial` = entitled
but never paid, `free` = everything else. This deliberately differs from the FCM *topic*
`tier_premium`, which counts trial users as premium. The topic definition has always been the
loose one; rules use the strict one.

⚠️ **`activeWithinDays` and `inactiveForDays` both exclude users with no activity data at
all.** Unknown is not treated as inactive — absence of evidence is not evidence of absence.

Exam scope is separate, on the rule itself: `examId: null` means every exam.

⚠️ **`examId: "upsc-cse"` also matches every user whose `active_exam_id` is NULL** — which is
the overwhelming majority, because only the app's exam switcher ever writes that column.
Treating NULL as "not UPSC" would exclude nearly everyone, so it is deliberately folded in.
The practical consequence: a UPSC-scoped rule and a null-scoped rule reach almost the same
people today. Scope to a state exam (`appsc-...`) and you get only users who actively switched.

This is also the ONLY place `active_exam_id` is read. It is deliberately not part of the
server's content-resolution chain — see [SME_EXAMS_API.md](./SME_EXAMS_API.md).

---

## 4.1 Two fields that are easy to misread

**`cooldownDays`** does NOT throttle a rule. Repeat suppression comes from the trigger's
dedupe key — an IST date for a daily reminder, an ISO week for the weekly report, an order id
for a payment. `cooldownDays` is a hint carried on the row for triggers that choose to read it;
today none do. Do not set it expecting it to stop a daily rule firing daily.

**`priority`** (lower wins) only matters when the per-user daily cap has to choose between two
rules competing for the same user on the same day. It does not affect send order or timing.
Seeded as PAYMENT 10, STREAK 20, everything else 50.

## 5. CRUD

```
GET    /sme/notification-rules?trigger=&exam=&enabled=
GET    /sme/notification-rules/:id
POST   /sme/notification-rules
PATCH  /sme/notification-rules/:id
DELETE /sme/notification-rules/:id        → archive (soft)
```

`POST` body: `triggerKey` and `name` required; everything else falls back to the trigger's
defaults.

> **A rule is always created disabled at 0%, whatever the body says.** Enabling is a separate,
> separately-audited call.

`PATCH` **merges** JSON blobs, it does not replace them. An omitted key keeps its stored
value — so a portal build that predates a field cannot silently delete it. (This repo has been
bitten by exactly that: a wholesale JSON replace dropped `priceInPaise` and the paywall
advertised the wrong price for two days.)

Two fields are deliberately **not** patchable:

- **`triggerKey`** is immutable. Repointing a rule's audience while keeping its copy, dispatch
  ledger and run history would make its analytics a lie. Create a new rule.
- **`isEnabled` / `rolloutPercent`** have their own endpoints, so turning something on is never
  something that rides along inside a copy edit.

`config` is validated against the trigger's schema: an unknown key is a **400 naming the
allowed keys**, and an out-of-range value is a 400 naming the bounds. Nothing is silently
dropped or clamped.

`DELETE` archives. Rules are never hard-deleted — dispatch rows and audit entries point at them.

---

## 6. Before you turn anything on

### 6.1 Dry run

```
POST /sme/notification-rules/:id/preview
```

Resolves the **real** audience and applies the **real** suppression chain. Sends nothing,
writes nothing.

```jsonc
{
  "matchedUsers": 3412,     // the trigger's predicate
  "afterPrefs":   3255,     // minus opt-outs and snoozes
  "afterCaps":    2891,     // minus the per-user daily cap
  "suppressed": { "rollout": 0, "optedOut": 157, "cooldown": 30,
                  "quietHours": 0, "cap": 364, "noDevice": 0 },
  "sample": [ { "userId": "...", "title": "Don't break your 12-day streak",
                "body": "...", "deepLink": { "type": "daily_task" } } ],
  "skippedReason": null,
  "error": null
}
```

**Read `matchedUsers` before enabling anything.** A number far larger than you expected means
the predicate is wrong, and you have learned that with nobody notified.

⚠️ Preview honours `rolloutPercent`. At 0% the `after*` counts are 0 — that is correct, it is
what would actually happen. `matchedUsers` is the number to size the audience by.

### 6.2 Test send

```
POST /sme/notification-rules/:id/test
{ "userIds": ["<id>", "..."] }        // max 10
```

Sends this rule's rendered copy to **those users only** and reports what each device did.

| Bypassed | Not bypassed |
|---|---|
| `isEnabled` | **The recipient list** — named ids only, max 10 |
| The schedule | **The global kill switch** |
| The audience predicate | |
| Quiet hours | |
| The daily cap (and it does not consume one) | |
| The cooldown / dedupe | |
| The category opt-out — but it is **reported** as `wouldSuppress` | |

```jsonc
{
  "isTest": true,
  "rendered": { "title": "...", "body": "...", "deepLink": { "type": "premium" } },
  "recipients": [
    { "userId": "...", "deviceCount": 2,
      "devices": [ { "platform": "ios" }, { "platform": "android" } ],
      "wouldSuppress": ["optedOut:content"] }
  ],
  "sent": 1, "failed": 0
}
```

`deviceCount: 0` is the answer to "why didn't I get it" — that account has no registered
device, and a real run would not have reached it either. Test sends are marked `is_test` and
are excluded from every analytic.

### 6.3 Enable, then ramp

```
POST /sme/notification-rules/:id/enable    { "rolloutPercent": 5 }
POST /sme/notification-rules/:id/disable
```

Bucketing is deterministic per (rule, user): **raising the percentage only ever adds people**,
it never reshuffles who already received it. Different rules bucket differently, so the same
unlucky 5% is not the test group for everything.

`enable` refuses (400) if the trigger is unavailable, if a scheduled rule has no `sendAtIst`,
or if the title or body is empty.

---

## 7. Guardrails

```
GET   /sme/notification-rules/settings
PATCH /sme/notification-rules/settings
```

```json
{
  "enabled": true,
  "shadowMode": false,
  "quietHours": { "start": "22:00", "end": "08:00" },
  "maxPerUserPerDay": 2,
  "priority": ["PAYMENT", "STREAK", "PROGRESS", "CONTENT"]
}
```

- **`enabled: false`** — the kill switch. Nothing evaluates, nothing sends.
- **`shadowMode: true`** — forces even *enabled* rules to shadow. The emergency freeze; you do
  not need it for a careful rollout, because disabled rules are shadow-run anyway.
- **`quietHours`** — no non-transactional sends inside this IST window. Wraps midnight.
- **`maxPerUserPerDay`** — **clamped server-side to 0–5.** A value outside that is corrected,
  not obeyed.

Transactional rules bypass quiet hours, the cap and the opt-out.

### 7.1 Two levers that are NOT in this API

**`NOTIFICATION_ENGINE_ENABLED=false`** (environment variable, set on the Fargate task).
Stops the scheduler from registering at all — no ticks, no shadow runs, nothing. This is the
ops-side brake, above the portal's kill switch, and it is the one to reach for if the engine
itself is misbehaving rather than a single rule. Changing it needs a redeploy, so
`settings.enabled: false` is the fast path.

**The boot-time coverage alarm.** `payment_success` and `trial_ending` used to be hardcoded
sends. If either has no enabled rule, every replica logs on startup:

```
[NOTIFICATION GAP] 'payment_success' has NO enabled rule. This notification shipped
before the engine existed and users expect it. ...
```

If you ever see that in the logs, users are paying and not being told. It also fires when a
rule is enabled but sitting at 0% rollout. Nothing in the portal surfaces it — it is a log
line, deliberately, because it must be visible to whoever is watching the deploy.

---

## 8. Did it work?

```
GET /sme/notification-rules/:id/runs?limit=20
GET /sme/notification-rules/:id/analytics?days=30
```

A run row exists for **every** execution, including shadow runs that sent nothing — mode,
counts, the suppression breakdown, a rendered sample, and `error` if the rule failed.

> A rule that suddenly matches 0 users is not necessarily "nobody qualified". Check `error`
> and `skippedReason` on the run row first.

```json
{ "windowDays": 30, "sent": 3120, "tapped": 412, "tapRate": 13.2 }
```

⚠️ **`tapped` means the user actually opened the notification.** It is not the same as
`isRead` on `/sme/analytics/notifications`, which only means they opened the in-app feed at
some point afterwards and badly over-counts. Judge a rule on `tapRate`.

---

## 9. What this cannot do yet

**`pib_today` is not implemented.** There is no PIB content in the product — no model, no
table, no publish date, no admin surface. The rule exists so the copy can be written and the
shape agreed, but it resolves no audience, `available` is `false`, and `enable` returns 400.

This is deliberate. Pointing it at reels or at library documents and hoping would ship a
notification that announces something that does not exist. **Tell us what PIB content actually
is and where it will live, and wiring it up is a small change** — nothing else about the rule
needs to move.

**Per-user send times** are not supported: a rule has one `sendAtIst` for everyone. The
scheduler ticks every 15 minutes, so that is the granularity.

**Language variants** are stored but not populated — every rule is English today.

---

## Common errors

| Status | When |
|---|---|
| 400 | Unknown `triggerKey`; unknown or out-of-range `config` key; `sendAtIst` on an event-driven trigger; enabling an unavailable trigger, a scheduled rule with no send time, or one with empty copy; `maxPerUserPerDay` outside 0–5 |
| 401 | Missing or wrong `x-api-key` |
| 404 | Unknown rule id; unknown `userIds` on a test send |

Every write is recorded in `sme_audit_log` — `NOTIFICATION_RULE_CREATE / UPDATE / ENABLE /
DISABLE / ARCHIVE / TEST` and `NOTIFICATION_SETTINGS_UPDATE` — with before/after snapshots.
`ENABLE` and `DISABLE` are separate verbs from `UPDATE` on purpose: turning a rule on is the
act that can reach thousands of devices, and it must be legible on its own.
