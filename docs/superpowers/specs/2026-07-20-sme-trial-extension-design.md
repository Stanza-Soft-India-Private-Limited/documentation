# SME Trial Extension — Design Spec

**Date:** 2026-07-20
**Status:** Implemented (backend)
**Scope:** Backend only (NestJS + Prisma + Neo4j mirror + Wylto sync). No app/web changes.

## Context

There was no way for SME to give a user extra free-trial time. The 14-day trial
window itself was not a first-class concept in the schema — it was a **literal
(`14 * 24 * 60 * 60 * 1000`, or `createdAt` + 14 days) duplicated across 6+
files**: `user.service.ts`, `sme-user.service.ts`, `premium-expiry.guard.ts`,
`payments.service.ts`, `auth/services/users.service.ts`, `wylto/wylto-status.util.ts`
(plus tests). Any change to the trial window, or any concept of "this user's trial
was moved," had no single place to land.

**Hard requirement:** existing users must be **completely unaffected**. The vast
majority of `UserAuth` rows will never be touched by this feature and must keep
resolving their trial end exactly as before, byte-for-byte.

## Decisions

- **Extension math:** `newEnd = max(now, currentTrialEnd) + N days`. Extending
  after a trial has already lapsed starts the N days from *now*, not from the
  stale past end — you can't get a shorter extension by waiting to click the
  button, and you can't stack negative/expired time.
- **Stacking, no cap:** repeat calls keep adding on top of the current effective
  end. No lifetime maximum — every call is independently audited
  (`sme_audit_log`, action `TRIAL_EXTEND`), so abuse is a review problem, not a
  code problem.
- **`days` bounds:** `1..365` (`ExtendTrialDto`, `class-validator` `@Min(1) @Max(365)`).
  Upper bound is a sanity rail, not a business rule — audit trail is the real
  guardrail.
- **Who's eligible:** anyone **not currently premium** — in-trial, trial-expired
  (never paid), or **churned** (previously paid, now lapsed). Explicitly
  includes churned users: a win-back extension is a first-class use case, not
  an edge case to block. Blocked: active paid subscribers (400 → "use grant
  premium instead") and non-standard lifecycle states (`SUSPENDED`, `LOCKED`,
  `ONBOARDING`, `INACTIVE` — 400).
- **No new Wylto vocabulary.** `derivePremiumState` (SME-facing) explicitly does
  **not** reuse `deriveWyltoStatus`'s label set — see the existing comment in
  `premium-check.util.ts` warning that coupling the two would let a marketing
  campaign rename silently change the SME portal's API contract. Trial
  extension respects that boundary: it only ever produces the existing five
  `PremiumState` values and the existing `WyltoStatus` values, never a new one.
- **Silent to the user.** The extension itself sends **zero** push/in-app
  notification — see the "no silent fallbacks" memory rule for *why we don't
  auto-decide messaging on the backend's behalf*: this isn't a fallback, it's a
  deliberate choice to let the portal control wording/timing. The portal is
  expected to pair the action with the existing `POST /sme/users/:id/notify`.
- **Both client-facing read endpoints get the new fields.** `GET /sme/users`
  (list) and `GET /sme/users/:id` (detail) both surface `trialEndsAt` +
  `trialDaysLeft`; detail additionally gets `trialExtended`. Keeping list and
  detail in sync avoids a portal UI that shows different trial state depending
  on which screen it's rendering.

## Chosen design: nullable `trial_ends_at` + single `getTrialEnd()`

Per project convention (add a column, never a new table):

```prisma
// SME-granted trial extension. NULL = default trial window (createdAt + 14d).
// When set, this is the authoritative trial end for every gate (getTrialEnd).
trialEndsAt DateTime? @map("trial_ends_at") @db.Timestamptz(6)
```

`src/common/utils/premium-check.util.ts` becomes the **single source of truth**:

```ts
export const TRIAL_WINDOW_MS = 14 * 24 * 60 * 60 * 1000;

export function getTrialEnd(user: { createdAt: Date; trialEndsAt: Date | null }): Date {
  return user.trialEndsAt ?? new Date(user.createdAt.getTime() + TRIAL_WINDOW_MS);
}
```

`trialEndsAt == null` is the entire legacy path — every one of the ~all existing
rows takes this branch and gets `createdAt + TRIAL_WINDOW_MS`, which is
byte-identical to what each of the 6 duplicated call sites computed before. The
6 consumers (`user.service.ts`, `sme-user.service.ts`, `premium-expiry.guard.ts`,
`payments.service.ts`, `auth/services/users.service.ts`,
`wylto/wylto-status.util.ts`) were switched to call `getTrialEnd()` instead of
re-deriving the literal, so there is now exactly one place that can compute a
trial end wrong.

`SmeUserService.extendTrial()` (`src/modules/sme/services/sme-user.service.ts`)
is the only writer of a non-null `trialEndsAt`:

```ts
const now = Date.now();
const currentEnd = getTrialEnd(user);
const newEnd = new Date(Math.max(now, currentEnd.getTime()) + dto.days * 86_400_000);

const shouldFlip =
  user.status === 'UNSUBSCRIBED' ||
  (user.status === 'SUBSCRIBED' && !isPremiumUser(user)); // lapsed-paid, not active-paid

await this.prisma.userAuth.update({
  where: { id },
  data: { trialEndsAt: newEnd, ...(shouldFlip ? { status: 'ACTIVE' } : {}) },
});
```

Guard checks before this runs (both 400, both in `extendTrial`):
1. `status === 'SUBSCRIBED' && isPremiumUser(user)` → active paid subscriber,
   reject (use grant premium).
2. `['SUSPENDED','LOCKED','ONBOARDING','INACTIVE'].includes(status)` → reject.

Best-effort side effects after the Postgres write (never block the response on
these — Postgres is the source of truth): `userRepo.updateUserTier(id, 'paid',
newEnd)` (Neo4j mirror, catch-and-log on failure) and
`wyltoSync.syncUserById(id)` (fire-and-forget, `.catch(() => {})`).

## The `derivePremiumState` / `deriveWyltoStatus` precedence rework

Both functions independently compute a label from `{status, premiumExpiresAt,
createdAt, trialEndsAt}`, and both needed the **same** precedence change: move
the active-trial branch *above* the everPaid/lapsed branch, and drop the old
`!everPaid` guard that used to gate it.

**Why:** before this feature, "ever paid" (`premiumExpiresAt != null`) and
"in-trial" were mutually exclusive by construction — a user who completed a
payment flow was always `SUBSCRIBED`, never `ACTIVE`, so an ever-paid user could
never also be inside a trial window. Trial extension breaks that invariant on
purpose: it revives a churned (ever-paid) user by setting `trialEndsAt` in the
future **and** flipping `status` back to `ACTIVE`. Without the reorder, such a
user would still read as `Churned` / `SUBSCRIPTION_LAPSED` (everPaid still
true) instead of `Trial` / `FREE_TRIAL_ACTIVE` — the whole point of "revive
them with full access" would be invisible to both the SME portal and Wylto.

`derivePremiumState` (`premium-check.util.ts`), branch order (first match wins):
1. active paid → `Premium`
2. `status === 'ACTIVE' && now < getTrialEnd(user)` → `Trial`
3. `everPaid` → `Churned`
4. `now >= getTrialEnd(user)` → `Trial Ended`
5. otherwise → `Downloaded`

**Why dropping `!everPaid` from branch 2 is safe for legacy rows:** the old
code additionally required `!everPaid` to enter the `Trial` branch. That guard
is now redundant, not load-bearing, for every row that predates this feature —
completing a payment always flips `status` to `SUBSCRIBED`, so there is no
existing (legacy) row that is simultaneously `status === 'ACTIVE'`, inside the
trial window, and ever-paid. The only way to reach that combination is the new
`extendTrial` write path itself. So removing `!everPaid` changes behavior for
**zero** pre-existing rows and enables exactly the one new case it was removed
for.

`deriveWyltoStatus` (`wylto-status.util.ts`) got the identical reorder, with an
extra comment nailing down why the `status === 'ACTIVE'` gate itself is
load-bearing (unrelated to this feature, but adjacent code touched during the
same change): it's what leaves `DOWNLOADED` reachable for a signed-in user
still `ONBOARDING`/`INACTIVE` inside the window (~226 live contacts) — removing
it would silently empty that Wylto status.

## The `PremiumExpiryGuard` Check-1 early-return (bug found in review)

`PremiumExpiryGuard` runs on every authenticated request and has always had two
checks: **Check 1** — if `premiumExpiresAt` is set and in the past, downgrade to
`UNSUBSCRIBED` (paid subscription expired); **Check 2** — if `status ===
'ACTIVE'` and past `getTrialEnd()`, downgrade (trial expired).

**The bug:** a churned user revived by `extendTrial` keeps their **old**
`premiumExpiresAt` — that field is left untouched on purpose (it's the historical
record of when they paid/expired, and `revokePremium`/`grantPremium` own it, not
`extendTrial`). That old value is, by definition, in the past for a churned
user. On the very next authenticated request after revival, Check 1 would see
`status !== 'UNSUBSCRIBED'` (true, they're `ACTIVE` now) **and**
`premiumExpiresAt < now` (also true, stale) and immediately downgrade them back
to `UNSUBSCRIBED` — insta-reverting the extension before the user ever got to
use it. This was caught in review, not by a test failure.

**The fix:** an early-return block inserted **before** Check 1, not a reorder of
Check 1's own condition:

```ts
if (
  userAuth.trialEndsAt != null &&
  userAuth.status === 'ACTIVE' &&
  now < getTrialEnd(userAuth)
) {
  return true;
}
// Check 1: explicit premium expiry (paid subscription expired) ...
// Check 2: trial expiry (status ACTIVE, never paid) ...
```

This is intentionally gated on `trialEndsAt != null` — i.e. it can only trigger
for a row an SME has touched. Every legacy row (`trialEndsAt === null`) skips
this block entirely and falls through to Check 1/Check 2 exactly as before, so
the fix cannot change behavior for any pre-existing user. It short-circuits the
whole guard (both checks) rather than special-casing Check 1 alone, because
Check 2 would otherwise also need the same "but not if trialEndsAt says
they're mid-extension" carve-out — one early-return covers both.

## Lapsed-SUBSCRIBED flip

`shouldFlip` in `extendTrial` also covers a second revival case beyond
`UNSUBSCRIBED`: `status === 'SUBSCRIBED' && !isPremiumUser(user)`. This is a
`SUBSCRIBED` row whose `premiumExpiresAt` has already lapsed but whose `status`
column hasn't caught up yet (it's only refreshed by `PremiumExpiryGuard` on an
authenticated request — a user who hasn't opened the app since expiring is
stuck showing stale `SUBSCRIBED` in raw storage). Without this arm, extending
such a user's trial would leave `status=SUBSCRIBED` with `isPremiumUser() ===
false`, which `derivePremiumState`/`isPremiumUser` would still read as
not-premium, so the extension would silently fail to grant access on the read
side even though `trialEndsAt` moved. Flipping to `ACTIVE` here makes the
revival correct on write, not just on the next guard-refreshed read.

## `eventAt` anchor change (Wylto)

`deriveWyltoStatus`'s `FREE_TRIAL_ACTIVE` branch used to anchor its
`eventAt` (used for "days since X" nurture flows) at `createdAt` unconditionally.
For an SME-extended trial, that would make a nurture sequence think a user who
was extended today is on "day 40 of a 14-day trial" — nonsensical. New anchor:

```ts
eventAt: Math.max(createdMs, trialEndMs - TRIAL_WINDOW_MS)
```

For a legacy user (`trialEndsAt == null`), `trialEndMs - TRIAL_WINDOW_MS ===
createdMs` exactly, so `max()` picks `createdMs` — **identical to before**. For
an extended user, this anchors to "the extension's effective start" (end minus
one window), so a day-1..14 drip counts from when the new trial began, not from
original signup.

**Caveat, not fixed here:** per the live Wylto contact-sync bug tracked
separately (`eventAt` never persists — the backend computes and sends it on
every PUT but has no local column to remember it, and Wylto's live API
currently has no way to read it back), this anchor only affects what we push to
Wylto on each sync call. Wylto's actual nurture-flow triggers today don't
consume `eventAt` from our payload in a way we've verified end-to-end, so this
is "correct as computed" rather than "verified as consumed." Not a regression
introduced by this feature — flagged so it isn't mistaken for one.

## ms-vs-calendar-math unification

Every trial-end computation in this feature (`getTrialEnd`, `extendTrial`,
`deriveWyltoStatus`) works in raw epoch milliseconds (`Date.getTime()` +
`86_400_000 * days`), not calendar day-arithmetic (no `date-fns addDays` /
timezone-aware month math). This is safe and intentionally simple because:

- The `IST timezone` project rule requires date-sensitive *user-facing*
  resolution (e.g. "which day is this for the user") to use IST, but trial
  extension isn't resolving "which calendar day" — it's adding a fixed
  duration to an instant, which is timezone-agnostic by definition.
  `now + N*86_400_000ms` and `IST-calendar-now + N days` land on the same
  instant either way.
- India (IST, UTC+5:30) has **no DST** — there is no day in the year where a
  24-hour block doesn't correspond to exactly one calendar day locally. A
  region with DST could have `+N days` in ms drift by an hour twice a year;
  IST cannot. So the ms-based math and calendar-based math never diverge here,
  and the simpler (ms) implementation was kept rather than pulling in a
  calendar-math dependency for a distinction that doesn't exist in this
  timezone.

## Client field semantics: EFFECTIVE trial end, not the raw column

`trialEndsAt` as returned by `GET /sme/users` and `GET /sme/users/:id`
(`formatUserSummary`) is **`getTrialEnd(user)`**, the effective end — not the
raw `trialEndsAt` database column. For a legacy user with a `NULL` column, the
API still returns a concrete ISO timestamp (`createdAt + 14d`), not `null`. The
column is `null`-able for storage/migration reasons; the API contract is never
supposed to force the portal to know about the NULL-fallback rule — it always
gets a resolved date (or `null` if the user isn't currently in a trial at all,
per the `inTrial` gate in `formatUserSummary`). `trialExtended` (detail only) is
the one field that *does* look at the raw column directly, precisely because
its job is to answer "did an SME touch this," which the raw NULL/non-NULL state
answers and the effective end does not.

## Rollout order (critical)

This feature adds a column to `UserAuth` — `user_auth.trial_ends_at`. Per
project convention, Prisma migrations are **not** run automatically; the ALTER
is handed to the user to run by hand:

```sql
ALTER TABLE "user_auth" ADD COLUMN "trial_ends_at" TIMESTAMPTZ;
```

**This must be applied to prod BEFORE the `mau` deploy that ships this code —
not after, not "close enough."** `JwtStrategy.validate()` loads the user via
`prisma.session.findUnique({ include: { user: { include: { userProfile: true
} } } })` — an `include`, which pulls **every current scalar column** of
`UserAuth` as defined in the deployed Prisma schema/client, on **every single
authenticated request** in the app (this is the auth hot path, not an
SME-only path). If the new Prisma Client (built from a `schema.prisma` that
already declares `trialEndsAt`) is deployed while the database still lacks the
`trial_ends_at` column, every `include`-based read of `UserAuth` — i.e. every
`/auth/refresh`, every JWT-guarded request across the whole app — throws
Postgres/Prisma **P2022 ("column does not exist")**. That is a full auth outage
for every user, not a degraded SME feature. Hand-apply the ALTER first, verify
via `information_schema` (per the "manual migrations drift" project rule — the
`_prisma_migrations` table is not trustworthy here), *then* deploy.
