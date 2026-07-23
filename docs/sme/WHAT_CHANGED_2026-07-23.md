# SME Portal — what changed, 2026-07-23

Everything below is **deployed and live on production** unless a line says otherwise.
Four documents in this bundle: one correction to a doc you already have, two new
API surfaces, and one UI proposal.

### Where to put these files

Drop them into the **same folder as the 2026-07-22 bundle** you already have. A few
cross-links point at documents from that bundle — `SME_PORTAL_API.md`,
`SME_NOTIFICATIONS_API.md`, `SME_ANALYTICS_FRONTEND_GUIDE.md` — which have **not changed**,
so your existing copies are current and the links will resolve once the files sit together.

`SME_USAGE_ANALYTICS.md` **is** included because it changed today: it previously said
notification time-to-read was underivable ("no `read_at` column"). That column now exists.
Replace your copy.

Confirm any endpoint is live with a keyless request — **401** means deployed and
guarded, **404** means not deployed:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  https://app.stanzasoft.ai/api/v1/sme/analytics/onboarding-funnel
```

Every endpoint named in this note was called against **production** on 2026-07-23 with a
real `x-api-key` and returned `200`. The numbers quoted throughout are the actual response
bodies, not illustrations.

---

## 0. Read this if you are pointing an AI agent at any of these docs

**Every document in this bundle that results in a screen now opens with a mandatory
`frontend-design` block**, tailored to that specific surface — the activity trail, the
insight endpoints, the feedback module, the user-detail page and the analytics guide. Each
one states what the screen actually *is*, who reads it and under what pressure, and carries
a paste-ready brief you can hand to a coding agent verbatim.

That framing is not decoration. These five surfaces have five different jobs and they fail in
different ways when built from the same admin template:

| Surface | What it is | Failure mode if templated |
|---|---|---|
| User detail / activity trail | a support-desk **answer sheet**, read with a customer waiting | operator can't get oriented in ten seconds; `api_usage` noise buries the story |
| Question quality | a content-team **work queue** | sorted by a score that is `0` for every row today; the one real defect is invisible |
| Release health | a release-owner **go/no-go** | the force-update number is never computed; a paywall spike reads as a broken build |
| Notification effectiveness | a growth **kill list** | rendered as a leaderboard; the caveat gets separated from the number |
| Onboarding funnel | an ops **diagnostic** | five equal-size stat cards, which communicate nothing |

**Do not let an agent skip the skill invocation.** Run `/frontend-design` before the first
component, not after a draft exists.

---

## 🔴 1. `SME_FEEDBACK_API.md` — CORRECTION. Read this first.

**You already have this file** (2026-07-22 bundle). **That copy is wrong.**

It said all responses ride a global `{ success, message, data, timestamp, path }`
envelope and that you should read `data`. **There is no such envelope.**
`ResponseInterceptor` exists in the codebase but is registered nowhere; the only
global interceptor is `ApiUsageInterceptor`. **Success bodies come back RAW** — a list
endpoint returns a bare JSON array, an object endpoint a bare object.

Error bodies *are* shaped, but by the exception filter: `{ success: false, message,
error, statusCode, timestamp, path, method }`.

**Client rule: branch on the HTTP status code, never on the presence of `success`.**

This is not theoretical — our own mobile app built envelope-decoding on that sentence
and the surveys inbox crashed on-device. **If you have already written an unwrapping
helper, that is the bug**, and it will look like "the API returns nothing".

Delete the old copy. No endpoint behaviour changed, only the documentation.

---

## 🆕 2. `SME_ACTIVITY_TRAIL_API.md` — per-user activity trail

The support/debug view: **"what happened to this user, end to end."**

- `GET /sme/users/:id/snapshot` — who they are, what they've paid, what's broken, now
- `GET /sme/users/:id/timeline` — 17 merged sources, cursor-paginated
- `GET /sme/users/:id/incidents` — errors auto-clustered ("at 14:22, 3 failures on the payment route")
- `POST /sme/users/:id/notes` — support notes pinned to the user
- `GET /sme/users/by-support-code/:code` — resolve the short code shown in the app

**Read its "What this data can and cannot tell you" section before designing anything.**
Several fields are deliberately honest about missing data and the UI has to render
those states rather than showing a zero.

---

## 🆕 3. `SME_INSIGHTS_API.md` — four new analytical surfaces

- `GET /sme/content/question-quality` — **a work queue for the content team.** Ranks items
  by how far their wrong-rate exceeds *chance*, not by raw difficulty. A well-formed MCQ
  cannot perform worse than guessing, so "worse than chance" flags a defect — a wrong key,
  or two defensible options. Also surfaces `contradictsKey`: when the most-picked option
  beats the key, the key itself is probably wrong. **Covers content-doc MCQs and
  simulations only — PYQ attempts are stored in Neo4j and are not included.**
- `GET /sme/analytics/release-health` — error rate and client-errors-per-user by app
  version × platform. Turns "should we force-update?" into a number. `app_version = NULL`
  is its own labelled pre-1.7 bucket, never dropped.
- `GET /sme/analytics/notification-effectiveness` — ⚠️ `isRead` means **"opened the in-app
  feed"**, NOT "opened the push". That caveat ships inside every response so the number
  cannot travel without it. Ranks waste by *wasted sends*, not read-rate.
- `GET /sme/analytics/onboarding-funnel` — signup → phone → onboarding → psychometric →
  first action, with drop-off and time-to-stage, plus the OTP/login failure signal over
  the same window.

Each of the four now carries a **"How to use this data"** section: the questions it
answers, the decision it should drive, what a good visualisation is, the action that
follows — and the real production payload it returns today.

---

## 🆕 4. `SME_USER_DETAIL_PAGE_GUIDE.md` — UI proposal

Replaces today's congested sidebar drawer on `/app-users` with a dedicated page.

> ### ⛔ If you use an AI coding agent for this, it MUST invoke the `frontend-design` skill FIRST
> Run `/frontend-design` **before writing any UI code** — before the first component, not
> after a draft exists. Do **not** let the agent reach for a default admin-dashboard
> template. Paste this with the doc when you brief it:
>
> ```
> Invoke the /frontend-design skill first, before writing any UI code.
> Then build the SME user-detail page per SME_USER_DETAIL_PAGE_GUIDE.md,
> using SME_ACTIVITY_TRAIL_API.md as the authoritative field contract.
> Responses are RAW JSON — there is no {success,data} envelope.
> Render every "unavailable"/"no data yet" state explicitly; do not show 0.
> ```

The other docs in the bundle now carry the same block, each with its own brief. See §0.

---

## 📊 What these endpoints actually return in production, today

Measured 2026-07-23 against `app.stanzasoft.ai`. Quoted so you know what shape to design
for — and because in most cases the *real* data looks nothing like a tidy demo fixture.
Question quality is legitimately near-empty today, and that is the state you will hand
over in.

### Onboarding funnel — `days=30`

| # | Stage | Reached | % of signups | Conv. from prev | Median time |
|---|---|---|---|---|---|
| 1 | Signed up | **2,057** | 100% | — | t₀ |
| 2 | Phone verified | **1,853** | 90.1% | 90.1% | 29s |
| 3 | Onboarding completed | **1,464** | 71.2% | 79.0% | *not derivable* |
| 4 | Psychometric resolved | **1,478** | 71.9% | 101% | 6.9m |
| 5 | First genuine product action | **1,100** | 53.5% | 74.4% | 6.9m |

**About half of all signups never take a single product action.** The two biggest losses are
onboarding completion (**−389**) and psychometric → first action (**−378**) — near-equal, so
there is no single villain to fix. Phone verification is *not* a problem: 90.1% at a median
of 29 seconds.

Two things a naive funnel UI gets wrong here, both covered in the doc: stage 4's conversion
is **101%** with a **negative** drop (14 users resolved the psychometric gate without
`onboarding_completed` — stages are current-state booleans, not an ordered journey, and
clamping it to zero would hide real drift); and **991 of the 1,100 "first actions" are the
psychometric test itself**, so the headline is thinner than it looks — render
`stages[4].detail.bySource`.

⚠️ The window is rolling and conversion is evaluated *as of now*. A run twenty minutes
earlier the same day gave 2,059 → 1,855 → 1,466 → 1,480 → 1,101. Never cache a funnel
integer as a fact; show `meta.conversionMeasuredAsOf`.

### Question quality — `days=90&minAttempts=20`

`6 items eligible · 0 worse_than_chance · 1 suspicious · 5 ok`. At the default threshold the
entire content bank yields **six** scoreable questions in ninety days, and `priorityScore` is
`0` on all six because the Wilson lower bound never clears the 75% guessing rate at n≈25.
A screen that leads with `priorityScore` will look broken while working perfectly.

The one flagged item is the argument for the whole endpoint — question
`00f30b23…`, Economy: **24 attempts, 83.3% wrong**, key **D** picked by 16.7%, option **C**
picked by **41.7%**, `flag: "suspicious"`, **`contradictsKey: true`**.

**`contradictsKey` is the highest-yield field in the payload and should be the default
sort/filter for the content team.** When the most-picked option beats the marked key, that is
the signature of a **wrong answer key**, not a hard question — and unlike `priorityScore` it
needs no sample size to be interesting. At `minAttempts=5` the same window returns **149
eligible items, 3 suspicious, and 13 with `contradictsKey: true`** — an actual week's work
queue. Ship the screen with a low `minAttempts` default and a `contradictsKey` filter chip.

### Release health — `days=30`

| appVersion | platform | activeUsers | errorRatePct | verdict |
|---|---|---|---|---|
| `1.7` | ios | **2** | 0% | `insufficient_data` |
| `null` → *"pre-1.7 (no x-app-version header)"* | `null` | **2,097** | **0.5%** | `insufficient_data` |

**99.9% of the active base (2,097 of 2,099) is on a build that predates the version header —
and that share IS the force-update number.** The old build is not on fire (0.5% error rate,
only 16 paywall 402s in 170,219 requests), so the case for forcing an update is about shipping
the header and the telemetry, not an emergency.

⚠️ Do **not** read `preHeaderSharePct` off one platform row. Pre-1.7 builds send no
`x-platform` header either, so they all land in the `platform: null` rollup where that field
is trivially `100`, while `ios` reads `0`. Compute the headline across rollups.

### Notification effectiveness — `days=60`

| type | sent | read | maturedSent | **maturedReadRatePct** | wastedSends |
|---|---|---|---|---|---|
| `trial_ending` | 1,992 | 23 | 1,869 | **1.2%** | **1,847** |
| `payment_success` | 13 | 10 | 13 | **76.9%** | 3 |
| `home` | 7 | 5 | 6 | **83.3%** | 1 |
| `reel` | 1 | 1 | 1 | **100%** | 0 |

**The highest-volume notification is the least-read by an enormous margin — and that is
exactly the kill-list this endpoint exists to produce.** `trial_ending` is 99% of everything
we send and 1.2% of it is ever seen: 1,847 wasted sends across 1,033 people. Every other
type, all transactional and low-volume, sits between 77% and 100%. The live
`worstPerformers` entry says so in a sentence you can render verbatim.

⚠️ **Field-name trap — this one already cost an afternoon.** `readRatePct` exists **only on
`byDay[]`**. On `byType[]` and `totals` the rate field is **`maturedReadRatePct`**, because
the only honest denominator is the matured sample. Reading `byType[].readRatePct` in JS or
`jq` returns `undefined`/`null` for *every* row regardless of how healthy the data is —
the key simply does not exist. That is a reading error, **not** an endpoint defect: verified
live, all four types returned non-null `maturedReadRatePct`. The field is legitimately `null`
in exactly one case, `maturedSent === 0` (everything still inside the maturation window), so
branch on `maturedSent`, not on the rate.

### Activity trail — verified end to end

Snapshot, timeline (17 sources, cursor-paginated), incidents and support-code lookup all
returned `200` against a real production account. Support-code decoding is as lenient as
documented: `54p3 6y4m` resolved identically to `54P3-6Y4M`. Incidents came back **empty**
for a healthy user — design that state, it is the common one. `POST …/notes` is live on the
same guard and was not fired against production because it is the only write.

---

## 🔧 Contract corrections in this revision

Found by re-verifying every documented field against the source and against live production
responses. **No endpoint behaviour changed** — these are documentation fixes, but two of them
will break a client that trusted the old text.

| Doc | Was | Is |
|---|---|---|
| `SME_ACTIVITY_TRAIL_API.md` | `premiumState: "PREMIUM"` | **`"Premium"`**. The closed set is `Premium` · `Trial` · `Trial Ended` · `Churned` · `Downloaded` — title case, one value with a space. Not SCREAMING_CASE, and deliberately *not* the Wylto CRM vocabulary. |
| `SME_ACTIVITY_TRAIL_API.md` | `route: "/quota/begin"`, `lastRequestRoute: "GET /tasks/today"` | **the stored route keeps the `/api/v1` prefix** — `"/api/v1/quota/begin"`, `"GET /api/v1/tasks/today"`. It is Express's `request.route.path` verbatim (still templated: `/api/v1/pyq/:id/reveal`). Strip `^/api/v[0-9]+` yourself if you group by module. |
| `SME_INSIGHTS_API.md` | `platforms[]` example omitted `platformLabel` | it is present, and for the null platform it reads `"unknown (no x-platform header)"`. |
| `SME_FEEDBACK_API.md` | list response described ambiguously as "`data`:" | it is this endpoint's own pagination wrapper `{ data, total, page, limit, hasMore }` — a **per-endpoint** shape, not the global envelope that does not exist. `GET /sme/users` uses the same wrapper. |

---

## 🐞 Two live production problems found while verifying this bundle

Reported, not fixed. Neither is a portal bug; both change what you will see on screen.

1. **`DIFY_APP_KEYS` has no `mentor` entry on production.** The single chat rating ever
   collected (2026-07-22) has `difyForwarded: false, difyError: "no_api_key"` — so **100% of
   chat thumb ratings have failed to reach Dify**. `GET /sme/feedback/chat/summary` reports
   it honestly as `difyForwardFailures: 1` of `total: 1`. Until the env var is set, the chat
   feedback loop is collecting data and dropping it, and `topChips` will stay empty.
2. **`login_*` events are arriving in tiny numbers already.** `auth_events` held 2
   `login_attempt`, 1 `login_success` and 1 `login_failed` on capture day — a handful,
   consistent with internal test devices rather than the field, since the emitting app build
   has not been released. Do not read `loginSuccessRatePct: 50` off a sample of two. The
   panel becomes meaningful once the release ships.

---

## What will be EMPTY at first, and why

Build these panels to distinguish **"no data yet"** from **"nothing happened"** — for the
next few weeks they mean very different things.

| Data | Availability |
|---|---|
| Request history (`api_usage`) | **Rolling 30 days only** — raw rows are purged nightly. Re-run `SELECT min(created_at) FROM api_usage;` rather than hardcoding a window. |
| `auth_events` (login/OTP failures) | Capture began **2026-07-23**. Earlier days are structurally empty — the funnel returns `null`, never `0`. |
| OTP failures | `otp_send_failed` now has a **server-side** emitter (live). `login_*` and `otp_verify_failed` are **mobile-only** and produce nothing until the next app release ships AND users update. |
| Client errors, paywall funnel, app sessions, push-permission state | **Mobile-only — nothing until the next app release.** |
| Chat content | Lives only in Dify. We store the conversation **pointer**; the timeline proves a conversation happened and lets you look it up, it does not show what was said. |
| Quota | Redis with a midnight-IST TTL. **Today only** — yesterday is genuinely gone. Response says `scope: 'today_only'`. |
| Streak | From Neo4j, which has known stale/duplicate nodes. Degrades to `available: false` with a reason — **render that, never a 0**. ⚠️ `available: true` with `currentStreak: 0` is a *real* zero and must look different from `available: false`. |
| Time-to-read on notifications | **Not derivable at all today** — the endpoint returns `timeToRead.derivable: false` with a reason. `read_at` now exists but is stamped by the same **bulk** `markFeedSeen` update as `isRead`, so it records "came back to the app", never "opened this push". Pre-migration rows are NULL. Do not synthesise it. |
| Chat-thumb forward to Dify | Live, but **failing 100% of the time** — `DIFY_APP_KEYS` has no `mentor` entry on production, so every rating is stored with `difyError: "no_api_key"`. `difyForwardFailures` will equal `total` until the env var is set. See the production-problems section above. |
| Report inbox | **`total: 0`.** Module shipped 2026-07-22; the mobile build that submits reports has not been released. |
| Chat feedback chips | Empty — chip collection ships with the same app release. |

All timestamps and day buckets are **IST**.

### Two null-vs-zero rules that cover almost every case here

1. **`null` / `available: false` / `derivable: false` means "we did not record it."** Render
   the words, never a `0`, never a green tick. Several responses carry the sentence to render
   (`unavailableReason`, `reason`, `interpretation`, `meta.caveats[]`) precisely so you do not
   have to write it yourself.
2. **A real `0` is data.** `streak.currentStreak: 0` with `available: true`, `incidents: []`
   on a healthy user, `errorRatePct: 0` on a build with 88 requests — all true statements.
   They must be visually distinct from case 1. If your UI renders both as an em-dash, an
   operator will tell a customer something false.

---

## Also changed, no portal work needed

- **Display names.** 325 accounts were named `"Apple User"` — Apple only returns a name on
  first authorization. That is now 0. Where a real name was derivable from the email we
  used it (10); the rest are **NULL**, so fall back to phone or email in your UI. **31
  accounts have neither a name nor a phone** (Apple private relay, unverified) — they are
  genuinely unidentifiable and should render as something neutral, not as blank.
- **A rate-limiting bug** meant every "N per hour" limit was silently "N ever". Fixed. If
  the portal ever saw an endpoint stop accepting requests permanently, that was why.

---

## File manifest

| File | Location |
|---|---|
| `WHAT_CHANGED_2026-07-23.md` | `docs/sme/` — this note |
| `SME_FEEDBACK_API.md` | `docs/sme/` — **replaces** your existing copy |
| `SME_ACTIVITY_TRAIL_API.md` | `docs/sme/` |
| `SME_INSIGHTS_API.md` | `docs/sme/` |
| `SME_USER_DETAIL_PAGE_GUIDE.md` | `docs/sme/` |

`docs/sme/README.md` indexes all of them, plus the docs you already had.
