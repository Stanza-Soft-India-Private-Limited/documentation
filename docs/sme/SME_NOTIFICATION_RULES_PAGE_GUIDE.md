# Notification rules — page guide

**Audience:** whoever owns the SME portal. **Feed this to the portal's coding session.**
**Scope:** a NEW screen. Nothing to migrate, nothing to remove.
**Contract:** [`SME_NOTIFICATION_RULES_API.md`](./SME_NOTIFICATION_RULES_API.md) — this doc does
not repeat it; it says what to build on top of it.

> **Invoke the `frontend-design` skill before building any of this.** Two controls on this page
> can reach ~4,800 devices, and the difference between a safe page and a dangerous one is
> almost entirely a design problem: making "enable" and "send to everyone" *feel* like two
> different acts.

---

## 0. The one thing to get right

**Enabling a rule and sending it are two separate acts, and the UI must make that obvious.**

A rule is enabled at a `rolloutPercent`. Default `0` — enabled and reaching nobody. That is not
a quirk to paper over with a sensible default; it is the safety property. A portal that ships a
single **Enable** button defaulting to 100% has removed the guardrail the backend was built
around.

Suggested shape: **Enable** puts the rule into a *ramping* state showing a percent stepper
(5 → 25 → 50 → 100) with the resolved audience size beside it, so the operator always sees
*"5% of 3,412 ≈ 170 people"* before committing. Never show a bare percentage.

---

## 1. Build the rule form from the API, not from a hardcoded field list

`GET /sme/notification-rules/triggers` returns every trigger with its `configSchema`
(`key`, `label`, `type`, `min`, `max`, `default`, `help`), its `vars`, and its defaults.

**Render the config section from `configSchema`.** Do not hardcode `daysBefore`, `inactiveDays`
and friends. Triggers will be added later, and if the form is generated they appear with no
portal change — which is the entire point of the design. A hardcoded form silently ignores new
knobs and sends nothing for them.

Type mapping: `int` → number input with `min`/`max` enforced client-side (the API also 400s),
`bool` → switch, `string` → text. Always show `help` — it is written for the operator, not the
developer.

**`kind` changes the form:**

| `kind` | Show | Hide |
|---|---|---|
| `SWEPT` | Send time (`sendAtIst`), config knobs | — |
| `EVENT` | — | **Send time.** It fires on a domain event; the API 400s if you send `sendAtIst` |

**`available: false`** → render the rule read-only with `unavailableReason` shown inline, and
disable the Enable button. Only `pib_today` is in this state today: there is no PIB content in
the product. Do not let an operator turn it on and wonder why nothing happens.

---

## 2. The copy editor needs the variable list beside it

Copy is `{{placeholder}}` templating, and the valid placeholders differ per trigger — they are
in `vars` on the trigger. Show them as insertable chips next to the title/body inputs.

⚠️ **An unknown or missing variable renders as an empty string, not as `{{name}}`.** So a typo
produces *"Your Premium ends in  days"* — silently, with no error anywhere. The chips are the
defence; a free-text box with no affordance will produce typos.

The `/triggers` response now also carries `sampleVars` — the values a test send renders with.
Use them to power a live preview of the copy as the operator types; it is the same data the
backend will use, so the preview cannot drift from the test.

Enforce title ≤ 120 and body ≤ 500 with live counters. The API truncates rather than rejecting,
so an over-long body is lost quietly at send time.

A live preview rendered with placeholder sample values is worth building — the operator should
see the sentence, not the template.

---

## 3. The pre-flight flow is the feature

Three steps, and the page should walk the operator through them in order. Do not present
Enable as reachable before Preview has been run at least once.

**3.1 Preview — `POST /:id/preview`.** Returns `matchedUsers`, `afterPrefs`, `afterCaps`, a
`suppressed` breakdown and a rendered `sample`. Sends nothing.

Show the funnel as a funnel, not four unrelated numbers — the campaign analytics page already
suffers from "four denominators with no bridge" and this is the same trap:

```
matched 3,412  →  after prefs 3,255  →  after caps 2,891
                  suppressed: 157 opted out · 364 daily cap · 30 already sent
```

⚠️ **The counts are `null` when they would be meaningless** — on a failed run, and on every
EVENT-kind rule (`triggerKind: "EVENT"`). Render `null` as "not applicable", never as `0`, and
**do not compute a reach estimate from it**. Showing *"100% of 0 ≈ 0 people"* beside a live
Enable button is the exact failure this prevents — it happened on the first build of this page,
downstream of a preview that had already reported an error.

For an EVENT rule, replace **the funnel** with the `skippedReason` text — there is no standing
audience to size.

🔴 **KEEP THE REACH STEPPER.** `rolloutPercent` is enforced per-user inside the event path, so
an EVENT rule enabled at **0% sends to nobody, ever** — it just never fires, with no error and
no run row to notice. What is impossible for an EVENT rule is the *estimate* ("≈ N people"),
because there is no cohort to count; the percentage itself is as real and as necessary as it is
for a scheduled rule. Show the stepper with no people-count beside it.

(An earlier revision of this doc said to hide the stepper. That was wrong and would have made
every enabled event rule silently dead.)

⚠️ **`matchedUsers` ignores rollout; the `after*` counts honour it.** At 0% rollout the
`after*` numbers are legitimately 0. Label `matchedUsers` as *the audience* and the rest as
*what would actually go out right now*, or the operator will think preview is broken.

⚠️ **`error` and `skippedReason` are the important fields.** A rule that matches 0 users is not
necessarily "nobody qualified" — it may have failed. Surface those two prominently; if `error`
is non-null the number below it is meaningless.

**3.2 Test — `POST /:id/test { userIds: [...] }`, max 10.** Returns per-recipient
`deviceCount`, per-device `platform`, and `wouldSuppress`.

Render `deviceCount: 0` as a warning, not a zero: it is the answer to "why didn't I get it" —
that account has no registered device and a real run would not reach it either.
`wouldSuppress` is **reported but not honoured** for a test send; label it as
*"a real run would skip this person because…"* so nobody reads a delivered test as proof the
rule reaches opted-out users.

**3.3 Enable + ramp** — see §0.

---

## 4. The list page

Columns: trigger, name, exam, category, channel, send time, and **one status** — not two
booleans. `isEnabled` and `rolloutPercent` together produce three meaningfully different
states, and showing them as separate checkboxes guarantees someone misreads the middle one:

| State | Means |
|---|---|
| **Off** | `isEnabled: false` — not sending. Still shadow-running on schedule |
| **Enabled, 0%** | On, reaching **nobody**. The default after enabling |
| **Live (N%)** | Actually sending to N% of the audience |

The API returns `isLiveNow` derived for exactly this, so the portal never re-implements the
rule. Use it.

Group by trigger. Multiple rules share a `triggerKey` on purpose — a T-7/T-3/T-1 ladder is
three rows — and a flat alphabetical list makes a ladder look like duplicates.

---

## 5. Run history and analytics

**`GET /:id/runs`** — every execution including SHADOW runs that sent nothing. This is where
the value is *before* a rule goes live: a week of shadow rows is the evidence for whether
enabling it is safe. Show `mode` prominently; a SHADOW row with `sent: 0` is success, not
failure.

**`GET /:id/analytics`** — `sent`, `tapped`, `tapRate`.

⚠️ **Do not mix this with `isRead` from `/sme/analytics/notifications`.** They measure
different things and `isRead` is much larger:

| Metric | Actually means |
|---|---|
| `tapped` (here) | The user **opened this notification** |
| `isRead` (elsewhere) | The user opened the in-app feed at some point afterwards |

`tapRate` is the honest number for "is this rule worth sending". `isRead` overstates
engagement badly. If both appear on one screen, label them or someone will compare them.

⚠️ **`tapped` will be 0 for every rule until the mobile release that reports taps ships.** Do
not present a 0% tap rate as a verdict before then — show "not yet available" instead, or the
first person to look will kill a rule that is working.

---

## 6. Settings — small screen, real consequences

`GET`/`PATCH /sme/notification-rules/settings`. Four controls:

- **Enabled** — global kill switch. Style it as the emergency control it is.
- **Shadow mode** — forces even live rules to shadow. Freezes all sending without touching
  individual rules.
- **Quiet hours** — two time inputs, IST. Wraps midnight (22:00 → 08:00 is normal).
- **Max per user per day** — **clamped 0–5 server-side**; the API 400s above 5. Enforce the
  range in the input so the operator never sees a rejection for a value the UI offered.

Say on the page that transactional notifications (payment succeeded, subscription ended) bypass
quiet hours, the cap and the user's opt-out. Otherwise the first support question will be why a
payment receipt arrived at 23:40.

---

## 7. What NOT to build

- **No "send now" button.** There isn't an endpoint, deliberately. Ad-hoc sends already exist
  at `/sme/notifications/{broadcast,segment}` and `/sme/users/:id/notify` — and those bypass
  preferences, quiet hours and the cap, which is exactly why they are separate from rules.
- **No `triggerKey` editor on an existing rule.** It is immutable; the API rejects it.
  Repointing a rule's audience while keeping its copy and history would make its analytics a
  lie. Offer "duplicate as new rule" instead.
- **No hard delete.** `DELETE` archives. Dispatch rows and audit entries reference the rule.
- **No per-user preference editing.** The four category toggles belong to the user, in the app.
  The portal can see the aggregate effect via `suppressed.optedOut` in a preview; it must not
  be able to switch someone's notifications back on.

---

## 8. Errors worth handling specifically

| Status | Cause | Show |
|---|---|---|
| 400 unknown config key | Form sent a knob this trigger doesn't have — usually a hardcoded form | The message names the allowed keys |
| 400 out of range | Config value outside `min`/`max` | Enforce in the input instead |
| 400 on enable | Unavailable trigger, missing `sendAtIst`, or empty copy | The message says which |
| 400 on `sendAtIst` | Set on an EVENT trigger | Hide the field for EVENT kinds |
| 404 on test | An unknown `userId` | Name the ids — they are usually a typo |

Every write is audited (`NOTIFICATION_RULE_*`, `NOTIFICATION_SETTINGS_UPDATE`) with
before/after snapshots. `ENABLE` and `DISABLE` are separate audit verbs from `UPDATE` on
purpose — surfacing "who turned this on, and when" is worth a column in any activity view.
