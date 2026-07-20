# Wylto cohort statuses + tags — rollout plan

Agreed source of truth: `PrepMonkey_Cohort Statuses and Tags.xlsx` (14 Jul 2026).
All decisions below were confirmed with the PM on 14 Jul.

## The one mechanic that drives this whole plan

A Wylto contact stores a status **ID**, not a name (verified live: a contact reads back
`"status": "mtWvyLtDSzbKXRsZf765"`). Therefore:

- **Renaming a status moves zero contacts.** The ID is unchanged, so all 845 contacts on
  `Trial` instantly display as `FREE_TRIAL_ACTIVE`. No backfill, no API calls, no split
  population, nothing to lose.
- **Creating a new status starts at zero contacts.** Contacts only arrive when the backend
  syncs each one across.

So: **rename wherever the mapping is 1:1. Create only what's genuinely new.**
The single exception is `Premium`, which must split 1→2 (`MONTHLY` + `ANNUAL`) — that one
cannot be a rename and must be create → backfill → delete.

Confirmed: **no live Wylto flows today**, so renames break nothing.

## Phase 0 — Wylto dashboard (do this first; blocks the deploy)

### 0a. Rename 4 (instant, no data movement)

**These 4 are the ones WE created (16 Jun). Renaming is free and proven safe — see below.**

| Current | Rename to | Contacts carried |
|---|---|---|
| `Trial` | `FREE_TRIAL_ACTIVE` | ~845 |
| `Trial Ended` | `FREE_TRIAL_EXPIRED` | ~327 |
| `Churned` | `SUBSCRIPTION_LAPSED` | ~11 |
| `Downloaded` | `DOWNLOADED` | ~226 |

> **Not `New Lead`.** It is Wylto's own built-in, Wylto-assigned, and the initial status.
> We never write it, so the `New Lead` / `NEW_LEAD` naming mismatch is harmless — and
> renaming it risks breaking whatever assigns it. Leave it alone.

#### Rename safety — PROVEN LIVE, 14 Jul (not an assumption)

Tested with a throwaway status + a contact on a reserved fictional number, since deleted:

| Check | Result |
|---|---|
| Contact still attached after rename? | **Yes** — same contact id, same status id `ip4n1j0OPK4zLjbiOKOt` |
| New name resolves to the same entity? | **Yes** — identical status id |
| Old name after rename? | **HTTP 400 — dead instantly** |

A contact stores the status **ID**, not the name, so a rename is a pure relabel and every
contact rides along. **But the old name dies the moment you rename it** — which is exactly why
the sync kill-switch must be OFF while you make these changes (step 3 below), or every
running sync 400s until the new code deploys.

### 0b. Create 3 (start empty, filled by the backfill)

- `FREE_TRIAL_EXPIRING_24HR`
- `SUBSCRIPTION_PURCHASED_MONTHLY`
- `SUBSCRIPTION_PURCHASED_ANNUAL`

**After creating, confirm `NEW_LEAD` is still first in the list.** Wylto's own note — "the
first status is initial status" — means the top entry is what every auto-created contact
defaults to. Our OTP send auto-creates contacts, so if a new status lands at position 1,
every new signup silently defaults into it.

### 0c. The 4 leftovers — decide AFTER the backfill, not now

`Warm Lead` · `Cold Lead` · `Non Active User` (109) · `Not Downloaded`

None are in the agreed list and the backend never writes them. Don't touch them yet: the
backfill moves every phone-verified app user onto a real status, which drains these on its
own. Whatever is still sitting on them afterwards is, by definition, a contact with no app
account. Then decide from real counts:

- **`Non Active User` — delete.** It is Wylto's own guess and the `NO_ACTIVITY_5DAYS` tag
  replaces it with a real timestamp. **Check whether a Wylto automation still assigns it** —
  if so it will fight our sync (Wylto sets `Non Active User` → we set `FREE_TRIAL_ACTIVE` →
  Wylto sets it again), and that ping-pong burns a PUT every cycle. Deleting the status kills
  the automation, which is the point.
- **`Not Downloaded` — delete** if it drains to 0. It is a stale default; we never write it.
- **`Warm Lead` / `Cold Lead` — ask Ashana.** Keep only if marketing actively stages leads by
  hand. Note that neither survives on anyone who installs the app: our sync overwrites them
  with the real derived status. That is intended, but it means manual lead staging is
  lead-only, not user-only.

### 0d. Delete — LAST, after the backfill

`Premium` — only once it shows **0 contacts** (the backfill moves its ~3 users to
`SUBSCRIPTION_PURCHASED_MONTHLY` / `_ANNUAL`).

### 0e. Tags — nothing to do

All 14 already exist and match the sheet exactly. Verified against the dashboard:
`iOS` · `Android` · `Web` · `Full time` · `Part time` · `Upcoming` · `ONBOARDED` ·
`APP_FIRST_LOGIN` · `PSYCHOMETRIC_COMPLETED` · `NO_ACTIVITY_5DAYS` · `OPTED_OUT` ·
`OPTED_BACK_IN` · `BLOCKED_BY_USER` · `PAID_USER_GONE_QUIET`

Casing is load-bearing on the ones with spaces: `Full time`, `Part time`, `Upcoming`.

### 0f. Ask Wylto

Enable the **inbound "on new Message" webhook** and give us the URL/secret. It is needed for
exactly one thing now: `OPTED_OUT` / `OPTED_BACK_IN`. Until it exists, those two tags stay
empty and **no flow may filter on them** (see Phase 3).

## Status precedence (first match wins)

A contact holds exactly ONE status. `eventAt` always carries the date belonging to the
*current* status, so a flow's "days since EventAt" counts from the moment they entered it.

| # | Status | Rule | `eventAt` |
|---|---|---|---|
| 1 | `SUBSCRIPTION_PURCHASED_ANNUAL` | premium active + latest paid order is ANNUAL | purchase date |
| 2 | `SUBSCRIPTION_PURCHASED_MONTHLY` | premium active (any other plan) | purchase date |
| 3 | `SUBSCRIPTION_LAPSED` | ever paid + entitlement lapsed | expiry date |
| 4 | `FREE_TRIAL_EXPIRING_24HR` | never paid + ≤24h left of signup+14d | trial end |
| 5 | `FREE_TRIAL_ACTIVE` | never paid + inside signup+14d | trial **start** |
| 6 | `FREE_TRIAL_EXPIRED` | never paid + past signup+14d | trial end |
| 7 | `DOWNLOADED` | fallback (signed in, none of the above) | signup date |
| — | `NEW_LEAD` | **never written by us** — Wylto assigns it | — |

Trial window confirmed still **14 days**, derived from signup date. Premium state is derived
from timestamps, not the `status` column (which goes stale for dormant users).

## Tag rules

The backend must send the **complete** tag vector on every write — Wylto's PUT replaces the
whole set and there is no GET-contact to merge against. A tag omitted here is a tag deleted.

| Tag | Rule | Source |
|---|---|---|
| `iOS` / `Android` / `Web` | has an active device on that platform | `DeviceToken.platform` |
| `Full time` / `Part time` / `Upcoming` | from app onboarding | `UserProfile.aspirantType` (`FULL_TIME`/`PART_TIME`/`UPCOMING`) |
| `ONBOARDED` | finished onboarding | `UserProfile.onboardingCompleted` |
| `APP_FIRST_LOGIN` | always true for any synced user | user row only exists after first login |
| `PSYCHOMETRIC_COMPLETED` | psychometric done | `UserProfile.psychometricTestCompleted` |
| `NO_ACTIVITY_5DAYS` | **not premium** + last active > 5 IST days ago | `UserAuth.lastActiveAt` (new) |
| `PAID_USER_GONE_QUIET` | **premium** + last active > 7 IST days ago | `UserAuth.lastActiveAt` (new) |
| `OPTED_OUT` / `OPTED_BACK_IN` | STOP / START reply, last-write-wins | inbound webhook (Phase 3) |
| `BLOCKED_BY_USER` | **never emitted** — no signal exists | manual only |

### Two deliberate consequences

**1. The two inactivity tags are mutually exclusive by construction** (PM decision, 14 Jul,
revising an earlier "emit both"). They split on *premium status*, not on timing, so a contact
can never carry both and **no flow filter is needed**. The alternative — emit both, then add
`Filter: not PAID_USER_GONE_QUIET` to the win-back branch — pushes a backend concern into
flow config, where forgetting it means messaging a paying customer with "come back and
subscribe". Keep the rule in one place. Consequence: a premium user inactive 5–6 days carries
neither tag, then picks up `PAID_USER_GONE_QUIET` on day 7.

**2. The segment tag can be briefly wiped.** If a lead taps "Full time" in WhatsApp, then
installs and verifies but has not finished onboarding, `aspirantType` is null → we send no
segment tag → Wylto's tag is replaced away. It returns the moment they onboard. Non-app
leads are never synced by us, so their button-tap tags are never touched.

## Schema — 3 columns, no new tables

```prisma
// UserAuth
lastActiveAt          DateTime? @map("last_active_at")           // Phase 1
whatsappOptedOutAt    DateTime? @map("whatsapp_opted_out_at")    // Phase 3
whatsappOptedBackInAt DateTime? @map("whatsapp_opted_back_in_at")// Phase 3
```

Nothing else. `WAITLIST_USERS` is handled by Broadcast (no lead table, no import script),
`DOWNLOAD_LINK_*` and `DAY_1..14` are rejected in favour of the flow builder, and
`APP_LAST_ACTIVE_DATE` is no longer a Wylto field — so `eventAt` is free for the status date
and we never touch the contact's `message` field.

**Migration ordering is load-bearing.** `JwtStrategy` loads the session with
`include: { user }`, so Prisma SELECTs every `user_auth` scalar on every authenticated
request. Ship the client before the columns exist → **every authed request 500s**. ALTER
first, verify, then deploy. Migrations are run by hand here.

```sql
ALTER TABLE "user_auth" ADD COLUMN "last_active_at" TIMESTAMP(3);
-- Mandatory: without it every existing user reads as "never active".
UPDATE "user_auth" SET "last_active_at" = COALESCE("lastLoginAt", "createdAt");
```

## `lastActiveAt` — what it means and what it costs

"Activity" = **app launch** (the profile endpoint the app calls on open). Stamped **at most
once per IST day**, so it is one DB write per active user per day, not one per request.

Because we no longer push the last-active *date* to Wylto, the sync hash does not change
daily. A PUT fires only when the derived status or tag set actually changes — i.e. on the day
`NO_ACTIVITY_5DAYS` flips on, and again when the user returns and it flips off. An active
user costs **zero** PUTs/day in steady state.

Hash: `sha256(name|email|phone|statusName|sortedTags|eventAt)`. `eventAt` is included so a
renewal (which moves the purchase date without changing the status) still pushes.

## Rollout order

1. **Verify the rename assumption first** (2 min, see below).
2. Apply the migration on prod, by hand. Confirm the column exists and auth is healthy.
3. Deploy with `WYLTO_CONTACT_SYNC_ENABLED=false` — sync is dead, so no writes and no 400s
   during the dashboard changes.
4. Wylto dashboard: rename 3, create 3, confirm `New Lead` is still first.
5. Flip `WYLTO_CONTACT_SYNC_ENABLED=true`.
6. Backfill all phone-verified users (dry-run first). No `--force` needed — the stored hashes
   were computed under the old vocabulary, so every contact differs and pushes once.
7. Confirm `Premium` shows 0 contacts, then delete it.
8. **Phase 3** (when Wylto enables the webhook): inbound STOP/START → `OPTED_OUT` /
   `OPTED_BACK_IN`. Until then those tags stay empty and no flow may filter on them.

### The rename test (do before step 2)

Renaming preserving the contact→status link is a strong inference from the API's behaviour,
not something Wylto documents. Cheap to verify:

1. Create a throwaway status `ZZ_TEST` in the dashboard.
2. We PUT one test contact onto it and read back its status ID.
3. Rename it to `ZZ_TEST2`.
4. Re-read the contact. Same ID + still attached ⇒ rename is safe. Proceed.
5. Delete `ZZ_TEST2`.

If it fails, fall back to create-new + backfill + delete for all four, which costs one extra
backfill pass and a temporary split population.

## Known limits (do not re-litigate)

- **`BLOCKED_BY_USER` cannot be automated.** Wylto exposes no block or delivery-failure
  webhook. Anything set by hand on an app user is wiped by the next sync.
- **`APP_DOWNLOADED` / `NO_LOGIN_3DAYS` are impossible.** Apple and Google report no install
  signal. First login is the earliest observable event — hence `Downloaded` = signed in, and
  `APP_FIRST_LOGIN` as a tag on every synced user.
- **Wylto has no custom fields.** One status, one group, one `eventAt`, one free-text
  `message`, N tags. That is the entire writable surface.
- **Unknown names return 400**, and a 400 does not trip the adapter's auth breaker — it
  retries on every request forever. Hence the frozen allowlist in code + the dashboard-first
  ordering.
