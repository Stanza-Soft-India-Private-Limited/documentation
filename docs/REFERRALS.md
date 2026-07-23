# Referrals / invite loop

**Status:** built, not deployed. Backend `src/modules/user/referral.*`, mobile
`features/referral/**` in `upsc_app`.

**Scope:** attribution only. Who invited whom, recorded once, defensibly.
**Rewards are deliberately NOT built** — see [Reward hook](#reward-hook-not-built).

---

## Why this exists

`rg 'referral|invite'` returned zero matches across the repo before this change:
there was no growth loop of any kind. UPSC aspirants prepare in groups — coaching
batches, hostel floors, college WhatsApp groups — so peer invitation is the
natural loop for this product rather than a bolt-on.

---

## The code format

```
PM-4K3M-97T
│  └──────┴─ 7 symbols of Crockford Base32 (35 bits), drawn from a CSPRNG
└─────────── literal "PM" prefix — a marker, not payload
```

* **Alphabet:** `0123456789ABCDEFGHJKMNPQRSTVWXYZ` — Crockford Base32, which
  excludes `I`, `L`, `O` and `U`. The first three cannot be misheard against
  `1`/`0` on a phone call; dropping `U` means a code can never spell an
  obscenity. Same alphabet as the support code, on purpose: one alphabet for
  every code a human in this product reads aloud.
* **Stored form == displayed form.** `user_auth.referral_code` holds
  `PM-4K3M-97T` verbatim, hyphens included, so nothing has to re-format it.
* **Input is parsed leniently.** `pm-4k3m-97t`, `PM 4K3M 97T`, `PM4K3M97T` and
  the bare payload `4K3M97T` all resolve; `O`→`0` and `I`/`L`→`1` aliases are
  applied. This matters because the code arrives retyped out of a WhatsApp
  forward, not pasted.
* **Collision space:** `32^7 = 34,359,738,368`. Against a user base in the low
  thousands the probability of any clash at all is ~1e-4 — and the mint path
  still retries against the `referral_code` UNIQUE index rather than trusting
  that arithmetic.

### Why a referral code can never be confused with a support code

`deriveSupportCode` (`src/modules/sme/services/sme-activity.service.ts`) emits a
code like `K3M9-7TQ2`: exactly **8** payload symbols, **no alphabetic prefix**,
derived deterministically from the first 40 bits of the user id. Its decoder
requires exactly 8 symbols after separators are stripped.

A referral code is `PM` + **7** symbols = **9** separator-free characters.

|                        | Support code    | Referral code            |
| ---------------------- | --------------- | ------------------------ |
| Rendered               | `K3M9-7TQ2`     | `PM-4K3M-97T`            |
| Separator-free length  | 8               | 9 (or 7 bare)            |
| Prefix                 | none            | literal `PM`             |
| Source                 | derived from id | CSPRNG                   |
| Resolvable by          | SME staff       | any authenticated user   |

The lengths are **disjoint**, so neither parser can accept the other's output:

* a referral code handed to `decodeSupportCode` → 400 *"must be 8 symbols"*,
  rather than silently resolving the wrong human;
* a support code handed to `normalizeReferralCode` → 400 *"That looks like a
  support code, not an invite code"* — it names the mistake rather than saying
  "invalid".

There is a second, independent guarantee: one code is derived and one is random,
so a single user's two codes are different strings even if the lengths ever
converged. Both properties are covered by specs in `referral.service.spec.ts`.

---

## Endpoints

All three are authenticated (`JwtAuthGuard` is a global `APP_GUARD`; none of
these are `@Public()`). **The subject is always `@CurrentUser()`** — no route
takes a user identifier in the path, the query or the body. That is what keeps
redeem from becoming a way to attribute somebody else's account.

Responses are **raw JSON**. `ResponseInterceptor` exists in this repo but is
never registered, so there is no `{ success, data }` wrapper on success. Errors
are rendered by `AllExceptionsFilter` as `{ success: false, message, ... }`.

### `GET /user/referral/code`

Returns the caller's own code, minting it on first call.

```json
{ "code": "PM-4K3M-97T" }
```

### `POST /user/referral/redeem`

```json
// request
{ "code": "PM-4K3M-97T" }

// 200
{ "redeemed": true, "referredAt": "2026-07-23T09:14:02.113Z" }
```

`code` is the **only** accepted field. The global ValidationPipe runs with
`forbidNonWhitelisted: true`, so any extra field is a 400.

| Status | When                                                              |
| ------ | ----------------------------------------------------------------- |
| 400    | malformed code, own code, would create a cycle, past the window   |
| 404    | well-formed code that belongs to nobody                           |
| 409    | already attributed (write-once), including losing a concurrent race |
| 429    | rate limit tripped (per-user or per-IP)                           |
| 503    | rate limiter unavailable — redeem fails **closed**, retryable     |

### `GET /user/referral`

```json
{
  "code": "PM-4K3M-97T",
  "referredCount": 3,
  "referrals": [{ "joinedAt": "2026-07-21T05:31:00.000Z" }],
  "referredAt": null,
  "hasRedeemed": false,
  "canRedeem": true,
  "redeemWindowDays": 14
}
```

**`referrals` carries dates and nothing else.** No names, emails, phone numbers
or user ids of referred users, ever. A referral list is a social graph; the
referrer has no right to the identities in it, and "who else from my coaching
batch signed up" is precisely the leak to avoid. The Prisma read is
`select: { createdAt: true }`, so a column added to `user_auth` tomorrow cannot
leak through this endpoint either. `referredCount` is exact; the `referrals`
array is capped at 100 so one very successful referrer cannot make the endpoint
slow for everybody.

---

## Abuse rules

Enforced in `ReferralService.redeem`, in this order. Every one has a spec in
`src/modules/user/referral.service.spec.ts`.

| # | Rule | Result |
| - | ---- | ------ |
| 0 | Rate limit (per-user, then per-IP) | 429 — or **503** if the limiter is down |
| 1 | Code parses | 400 — never a 500 |
| 2 | **Write-once** — `referred_by_user_id` already set | 409 |
| 3 | **Recency window** — within 14 days of signup | 400 |
| 4 | Code resolves to a user | 404 |
| 5 | **No self-referral** | 400 |
| 6 | **No cycles** (direct and transitive) | 400 |
| 7 | Conditional write still matched | 409 |

Rules **6 and 7 run inside one interactive transaction**, guarded by a
transaction-scoped Postgres advisory lock — see [Cycles](#cycles).

### Write-once

`referred_by_user_id` is set exactly once and never rewritten. Re-attribution is
literally how reward farming works: redeem A's code, collect, redeem B's, collect
again. Two defences:

* the explicit check at rule 2, and
* the write itself is a compare-and-set:
  `updateMany({ where: { id, referredByUserId: null }, ... })`. Two concurrent
  redeems cannot both land — the loser gets `count: 0` and a 409.

Rule 2 runs **before** the code is looked up, so an already-attributed account
cannot use redeem as a code-existence oracle either.

### Recency window — 14 days, measured from `user_auth.created_at`

Chosen for four reasons:

1. **It matches the honest shape of the loop.** A friend is told about the app,
   installs it, and enters the code either during onboarding or the next time the
   two of them speak. Two weeks spans two weekends and two typical study-group
   meets, so it covers "remind me your code?" without inventing a second
   attribution mechanism.
2. **It is longer than the 7-day free trial.** A shorter window would push users
   to redeem before they have decided the app is worth recommending — the wrong
   incentive for the reward that will follow.
3. **It kills retroactive farming outright.** Every user already in the database
   when this ships is older than 14 days, so nobody can walk the existing base
   and attribute it to themselves. Attribution starts, permanently, with
   genuinely new signups.
4. **Anything materially longer is not a referral.** At 30 or 90 days you are
   attributing a user to somebody they may have met *after* they were already
   active. That is a coupon, not a referral.

Measured from `created_at` rather than install or first launch because that is
the only signup timestamp the server actually owns.

The window is also surfaced as `redeemWindowDays` in the status response and
named in the 400 message, so no client hardcodes `14`.

### Cycles

`referred_by_user_id` forms a forest, so a cycle can only occur if the redeemer
already sits **above** the referrer in the chain. `cycleVerdict` walks up from the
referrer (with a seen-set) and rejects if it reaches the redeemer. This catches
both:

* the direct case — A refers B, B tries to redeem A's code; and
* the transitive case — A → B → C, C's code offered to A.

The walk tolerates a missing row mid-chain, because `referred_by_user_id`
deliberately carries **no foreign key** so attribution survives the referrer hard
deleting their account.

#### The check and the write are ONE transaction

The cycle check is a predicate over **other** rows; the write touches only the
caller's. The compare-and-set at rule 7 therefore cannot protect it. A and B, both
unattributed, redeeming each other's codes milliseconds apart, each walk a chain
that is still pre-write, each see no cycle, and each write a **different** row —
both succeed, and `A.referred_by = B` **and** `B.referred_by = A`. The forest
invariant is gone, and the result is exactly the two-account structure the rule
exists to prevent once a reward attaches.

So rules 6–7 run inside one `prisma.$transaction`, whose first statement is
`pg_advisory_xact_lock(...)`. Redeems serialise against each other, the walk
observes the state the write commits on, and the lock is released by commit *or*
rollback (including on a thrown 400). A single global lock beats the alternatives
here: the rows needing protection are a data-dependent ancestor chain discovered
as the walk proceeds, so there is no fixed set to `FOR UPDATE` and no safe lock
ordering; `SERIALIZABLE` would work but pushes a 40001 retry loop into a path that
must never fail confusingly. Contention is a non-issue — redeem happens at most
once per account lifetime and is capped at 8/hr.

#### "Too deep" is not "cycle"

The walk is depth-capped (64) purely so it terminates. Hitting the cap means *we
did not finish looking*, *not* that a loop exists, and the two are reported
separately: `clean` / `cycle` / `too_deep`. A `too_deep` redeem is **allowed**,
with a `WARN` log naming the referrer.

This used to be a cap of **10** that returned "cycle" — so an eleven-deep chain,
which is what a *working* growth loop produces, made every new redeemer of the
deepest code get `400 "You already invited this person, so they cannot invite you
back."` They hadn't. A false rejection with a false explanation at exactly the
moment the loop starts working is far worse than the residual risk of allowing a
cycle above 64 hops — which requires 64 real accounts, each redeeming inside its
own 14-day window, and is loudly logged when it happens.

### Rate limiting

Redeem takes an opaque code and tells you whether it exists — an oracle. It uses
the existing `RateLimitService` (`src/common/services/rate.limiting.service.ts`),
following the `DiagnosticsService` pattern, on two independent keys:

| Key | Limit |
| --- | ----- |
| `user:referral:redeem:<userId>` | 8 / hour |
| `user:referral:redeem:ip:<ip>`  | 30 / hour |

Unlike diagnostics, a tripped limit **throws 429** rather than silently dropping —
a user typing their friend's code deserves to be told to wait, not told the code
is wrong. The IP is read from the transport (`clientIp`), never from the body.

**The per-user limit is the defence.** At 8 attempts/hour against `32^7`,
enumerating the space would take ~490,000 years per account.

> ⚠️ **The per-IP limit does not stop a farm.** `clientIp()` returns the first
> `x-forwarded-for` hop; `main.ts` never calls `app.set('trust proxy', …)`, so
> behind the ALB that header is the only signal and it is **client-controlled**.
> An attacker who rotates the header simply never collides with their own bucket,
> and `REDEEM_IP_LIMIT = 30/hr` never trips. It bounds an unsophisticated script
> and costs nothing, so it stays — but it must not be described as a ceiling on
> account rotation, because it isn't one. Setting `trust proxy` is a global change
> affecting every consumer of `request.ip` and is tracked separately.

### The limiter fails CLOSED here — and only here

`CacheService.increment` swallows every Redis error and returns `0`, so
`RateLimitService` computes `allowed: 0 <= limit` → **always true**. During a
Valkey outage — which has happened in production; it is the same swallowed-error
path that made `storeOtp` silently no-op — every rate limit in the app quietly
stops limiting.

That default is deliberate and stays: denying somebody an access right because
Valkey blipped is worse than briefly unmetered traffic, and `CacheService` /
`RateLimitService` are shared by many callers. Redeem is the exception, because
the limiter is not a nicety here — it is the *only* thing standing between an
attacker and an unlimited code-existence oracle.

So `ReferralService.enforceRedeemRateLimit` detects the degraded signal locally
and returns **503** (`error: "ReferralRateLimiterUnavailable"`), retryable, with a
message that does not read as "you hit the limit". The signal: `increment` returns
`>= 1` for every real call, so on a healthy limiter `remaining` can never equal
`limit` — `remaining >= limit` means the counter came back `0`, i.e. the swallowed
error branch. No global behaviour changes.

---

## Minting

`referral_code` starts NULL for every existing user, and it is minted on first
read rather than backfilled — a backfill would generate thousands of codes nobody
asked for, and a data migration is not something this change owns.

The mint is a compare-and-set on `referralCode: null`, so two concurrent first
reads cannot produce two codes: the loser re-reads and returns the winner's. A
code, once minted, is **never rotated** — it is already in somebody's WhatsApp
history.

---

## Deep link

`https://app.prepmonkey.com/open/invite/<code>` opens the app on the Invite
screen with the code prefilled. Handled by `DeepLinkMapper` in the KMP app
(`DeepLinkTarget.Invite`), consumed in `MainTabsScreen` like every other target —
so it only fires once auth has resolved, and the redeem call is made by an
authenticated user as usual.

**Web still needs the route.** `prepmonkey-web`'s `/open/*` handler has no
`invite` case yet, so the link currently falls through to that app's default
behaviour on desktop/no-app-installed. Nothing breaks — the mobile half works —
but the web fallback page is a follow-up outside this change's file ownership.

---

## Reward hook (NOT built)

Attribution is the hard, abuse-prone half and it is what this change delivers.
The natural follow-up is a reward on successful attribution. The pieces already
exist:

* `user_auth.trial_ends_at` is already the field that gates trial access, and
* `POST /sme/users/:id/trial-extension` already grants an extension through the
  SME portal.

So a reward hook would be: on a successful `redeem`, extend `trial_ends_at` for
**both** the redeemer and the referrer by N days, reusing that same grant path.

Things to settle before building it:

* **Both sides or one?** Two-sided rewards drive the loop harder but double the
  farming payoff.
* **A cap per referrer.** Without one, a single user with a large WhatsApp
  audience can extend their trial indefinitely.
* **A qualification bar.** Rewarding on signup alone rewards installs; rewarding
  on, say, first completed practice session rewards actual users. The status
  endpoint already returns join dates, so a qualified-vs-raw split is cheap to
  add later.
* **Idempotency.** Write-once attribution makes the reward naturally
  once-per-redeemer, but the *referrer* side accrues repeatedly and needs its own
  ledger (there is no `referral_rewards` table today).

---

## Files

| Path | What |
| ---- | ---- |
| `src/modules/user/referral-code.util.ts` | Format, mint, lenient parse |
| `src/modules/user/referral.service.ts` | Attribution + every abuse rule |
| `src/modules/user/referral.controller.ts` | The three routes |
| `src/modules/user/dto/redeem-referral.dto.ts` | The one-field redeem body |
| `src/modules/user/referral.service.spec.ts` | 40+ specs, one per rule |
| `src/modules/user/referral.controller.spec.ts` | Subject-from-JWT, raw body |

Schema (`referral_code`, `referred_by_user_id`, `referred_at` on `user_auth`)
was added separately and is not owned by this module.
