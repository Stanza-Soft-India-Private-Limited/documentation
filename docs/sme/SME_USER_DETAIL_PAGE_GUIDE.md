# SME User Detail — Page Design Proposal

> ## ⛔ BEFORE YOU WRITE ANY UI CODE — INVOKE THE `frontend-design` SKILL
> If you are a Claude/coding session building this page, your **first action** is to invoke
> the **`frontend-design`** skill (`/frontend-design`). Do this **even if no one asked you to**
> and even if the surrounding task didn't mention design. Every panel, table, timeline row,
> and empty state described below must be designed through that skill — do not reach for a
> default admin-template layout. This is a hard requirement, not a suggestion.
>
> Design for what this screen actually is: **the support desk's answer sheet.** An internal
> operator has a PrepMonkey user on the phone or in a WhatsApp thread *right now*, saying
> something went wrong. This page has to make the operator competent in about ten seconds —
> who this is, whether they're paid, whether something is broken — and then let them dig.
> Audience = internal support/ops staff under time pressure, not consumers, not analysts.
> The job of the screen is *fast situational awareness*, then *drill-down*.

This document is the **UX intent + information architecture** for replacing today's
user-detail experience. It is the companion to
[`SME_ACTIVITY_TRAIL_API.md`](./SME_ACTIVITY_TRAIL_API.md), which is the authoritative
contract for every field named here. Read that first — especially its
"What this data can and cannot tell you" section, because half the design decisions below
exist to render those limitations honestly.

---

## Why replace the drawer

Today the user detail is a **sidebar drawer over the `/app-users` list**. That was fine when
it showed six fields. It now has to carry entitlement, orders, quota, streak, activity counts,
device/version info, incidents, a multi-source timeline and staff notes. In a drawer that
means:

- **No room for the timeline**, which is the whole point of the new API — a chronological
  narrative needs vertical space and a scroll of its own, not a 420px column already scrolling.
- **Nothing is linkable.** An operator cannot paste "the page for this user" into a ticket or
  a colleague's DM. Every handoff becomes "search for their email, then click the third row."
- **The list underneath is dead weight.** Once you're diagnosing one user, the roster behind
  the scrim is noise competing for the same scroll.
- **Filters can't coexist with content.** The timeline needs source filters, a date window and
  pagination; the drawer has no place to put them that isn't stealing from the content.

**Proposal:** a dedicated route — `/app-users/:id` — with the roster remaining at
`/app-users` and each row linking into it. Keep a lightweight *peek* affordance on the list if
the team likes it (hover card / quick-look), but the diagnostic surface is a real page with a
real URL.

**Two entry points, both must work:**
1. From the roster (click a row).
2. From a **support-code lookup** in the global header — the operator types the code the user
   just read out over the phone, and lands directly here. This is the fast path and it should
   feel like the primary one. See `SME_ACTIVITY_TRAIL_API.md` §5.

---

## Information hierarchy

Top-down by *how fast an operator needs it*. Everything above the fold answers "who and what
state"; everything below answers "what happened".

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. SNAPSHOT HEADER      identity · entitlement · quota · streak   │  ← GET :id/snapshot
│                         last seen · devices/versions · push       │
├──────────────────────────────────────────────────────────────────┤
│ 2. INCIDENTS            "what went wrong" — grouped failures      │  ← GET :id/incidents
├──────────────────────────────────────────────────────────────────┤
│ 3. TIMELINE             everything, reverse-chronological,        │  ← GET :id/timeline
│                         source filters + date window + paging     │
├──────────────────────────────────────────────────────────────────┤
│ 4. NOTES                staff notes — write + read                │  ← POST :id/notes
└──────────────────────────────────────────────────────────────────┘
```

Fire all three GETs **independently and in parallel** on mount. The snapshot is the fastest and
must not wait on the timeline; the incidents call must not be blocked by a slow Neo4j read
inside the snapshot. No panel blocks another.

---

## 1. Snapshot header — `GET /sme/users/:id/snapshot`

The identity + state band. It should be readable without scrolling and without interaction.
Think "patient chart header", not "dashboard".

**Left — identity**
`name` (or a graceful fallback when `null` — many users have no name), `email`, `phoneNumber`
with a `phoneVerified` marker, `status`, `provider`, `createdAt` as "member since".
The **`supportCode`** belongs here, rendered large and monospaced with a click-to-copy — it is
what the operator reads back to the user to confirm they're looking at the right account.
If `supportCode` is `""` (non-UUID id), render nothing at all — never an empty box.

⚠️ **Do not derive premium from `status`.** `status: "ACTIVE"` is the *trial* state.
`entitlement.isPremium` / `entitlement.premiumState` are the authoritative answer.

**Centre — entitlement (the single most-asked question)**
A clear premium/trial/free state chip driven by `entitlement.premiumState`, with
`premiumExpiresAt`, `subscriptionSource` (Apple vs Razorpay changes what the operator can even
do), and — when in trial — `trialEndsAt` + `trialDaysLeft`. Beneath it, `recentOrders[]` (last
5) as a compact list: status, amount, currency, plan, source, and **`premiumGrantedAt`**.
`premiumGrantedAt: null` on a `PAID` order is the exact signature of "paid but not activated" —
make it visually loud, because that is the complaint that brought them here.
Amounts are already in major units. **Do not divide by 100.**

**Right — today's state**
- **Quota** — `quota.features` rendered generically off each entry's `type`
  (`daily`/`lifetime` show `used`/`limit`/`remaining`; `premium_only` shows a lock;
  `unlimited` shows "unlimited"). Label the block **"Today (IST)"** and say
  *"resets at midnight IST — no history is kept"*. Never draw a trend line; there is no series
  to draw. When `available: false`, show the `unavailableReason` in place of the numbers.
- **Streak** — `currentStreak` / `maxStreak` / `lastActiveDate`. When `available: false`,
  render **"Streak unavailable"** plus the reason. **Never a zero.** "0-day streak" and "we
  couldn't read the graph DB" are different sentences to an operator, and only one of them is
  the user's fault.

**Bottom strip — reachability**
- `lastSeen` as five relative timestamps with their absolute values on hover:
  `lastLoginAt`, `lastActiveAt`, `lastSessionAt`, `lastRequestAt` (+ `lastRequestRoute`),
  `lastClientEventAt` (+ `lastClientEventType`). They disagree on purpose — different sources —
  so label each rather than picking one "last seen".
- `clients.appVersions[]` as version+platform chips. More than one entry means an upgrade or a
  second platform; that is diagnostic gold when a bug is version-specific.
- `clients.distinctDeviceCount` — render `50+` when `deviceCountCapped` is `true`, never `50`.
- `clients.pushTokens[]` — platform, active/inactive, and a **three-state** permission
  indicator: `true` granted, `false` **denied** (this is why your notification "succeeded" and
  the user saw nothing), `null` **unknown — client build predates the field**. Three states,
  three visual treatments. Collapsing `null` into "denied" will send an operator down a wrong
  path.

**Activity counts** (`activity.counts`) sit as a compact row of small numbers — one per source,
labelled in human words ("Requests", "Questions answered", "Notifications", "Payments"…). Each
count is a **link that scrolls to the timeline with `?sources=<key>` applied**. That is the
whole navigation model of this page: the header is a set of doorways into the timeline.
Always show the window (`activity.windowDays`, `from`, `to`) next to them — a count without its
denominator is not trustworthy.

**States**
- *Loading:* skeleton the four regions independently.
- *Empty:* a brand-new user has zeroed counts and null last-seen. Write it as
  "Signed up 3 minutes ago — nothing recorded yet", not as an error.
- *Partial:* quota and streak degrade individually (`available: false`). The rest of the header
  must render normally. Design the degraded tile as a first-class state, not an error toast.
- *Error (404):* the whole page is meaningless — show a "No such user" state with a link back
  to the roster and to the support-code lookup.
- *Error (5xx/401):* state what failed and offer retry, in the console's voice. A 401 means the
  API key is wrong/missing — say that explicitly rather than "something went wrong".

---

## 2. Incidents — `GET /sme/users/:id/incidents`

**This is the shortcut that makes the page worth building.** Instead of scrolling a timeline
hunting for red, the operator sees *"at 14:22 IST, 3 failures on POST /payments/verify"*.
Place it directly under the header, above the timeline.

Each incident row: `startedAtIST` + `istDate` as the time anchor, the server-rendered `title`
as the headline, `failureCount`, a `severity` treatment, and the `byLabel[]` / `byStatus[]`
tallies as compact chips. Expanding a row reveals `samples[]` — capped at **5 rows**, which
arrive in the **exact same item shape as timeline rows**, so build one `<ActivityItem>`
component and reuse it here. When `failureCount > samples.length`, say "showing 5 of N".
That shared component is a design constraint worth honouring; it keeps the two panels reading
as one system.

Give the operator two controls and no more:
- **Sensitivity** — `gapMinutes` (default 5, max 60). Frame it in plain words
  ("group failures within 5 minutes"), not as a raw parameter.
- **Window** — shared with the timeline (see below); do not give incidents its own date picker.

**Honesty requirements**
- `meta.rowCapHit: true` → at least one source hit its own `meta.perSourceRowCap` (250 each
  for `api_usage` and `auth_events`, i.e. long before `rowsScanned` reaches 500) and there are
  almost certainly more failures. Show a persistent, quiet banner naming
  `meta.truncatedSources`: *"Showing the most recent 250 request failures in this window."*
  `hasMore` alone will not tell you this — check the flag.
- `meta.retention` (on the timeline, incidents and `snapshot.activity`) → when
  `windowExceedsApiUsageRetention` is true, render `retention.note`. A 90-day window is
  genuinely empty of `api_usage` past day 30; that is the purge, not an idle user.
- Paginate on `nextCursor`; if you need the anchor row yourself, use `incident.oldestRowId`.
  **Never** read it off the last element of `samples` — that array is truncated to 5.
- Incidents are clustered **within a page**. A burst spanning a page boundary appears split.
  Say so in a tooltip on the pagination control rather than trying to stitch clusters together
  client-side; the fix is a larger `limit`, and the doc says so.
- `id` changes when `gapMinutes` changes — never persist or deep-link an incident id.

**States**
- *Loading:* two or three skeleton rows. Don't block the timeline on this call.
- *Empty:* **this is a good outcome and should read like one** — "No failures in the last 30
  days." Do not draw an error-coloured empty state for the absence of errors.
- *Empty-because-nothing-captured:* if the snapshot shows `api_usage` and `auth_events` counts
  of zero, the honest message is "No request history for this user in this window" — different
  from "no failures". See the capture caveats below.
- *Error:* inline retry on this panel only; the rest of the page stays alive.

---

## 3. Timeline — `GET /sme/users/:id/timeline`

The narrative. Reverse-chronological, merged across 17 sources.

**Layout.** A single vertical stream grouped by **IST calendar day** (use `istDate` semantics —
convert `at` at UTC+05:30; do not group on the UTC date, or an operator's evening events land
on tomorrow). Each row: a time, a source marker, the server-supplied **`title`** as the
one-liner, and a severity treatment. **Use `title` — do not reimplement 17 per-source
formatters in the portal.** Expanding a row shows the `data` object as a labelled key/value
detail, not a raw JSON dump (JSON is acceptable as a "copy raw" affordance for engineers).

**Controls, in one persistent bar:**
- **Source filter** — multi-select over the 17 canonical sources, grouped for humans:
  *Auth & client* (`auth_events`), *Requests* (`api_usage`), *Money* (`orders`, `payments`,
  `payment_events`, `refunds`), *Study* (`user_question_attempts`, `simulation_attempts`,
  `user_document_progress`, `custom_tasks`, `user_content`, `psychometric_test_results`),
  *Notifications* (`notification_history`), *Staff* (`sme_audit_log`), *Feedback*
  (`feedback_reports`, `survey_responses`, `chat_message_feedback`).
  Send canonical names. The aliases (`webhook_events`/`webhooks` → `payment_events`,
  `notes`/`staff` → `sme_audit_log`) exist for hand-written URLs and for the count-tile
  deep-links in the header — accept them on inbound query strings, but always **normalise to
  the canonical name** in your own state, because that's what `meta.sources` and every item's
  `source` come back as.
- **Window** — `days` presets (7 / 30 / 90) plus an explicit `from`/`to`. Bare `YYYY-MM-DD` is
  interpreted as an **IST** calendar day by the server, which is what an operator means.
- **Page size** — optional; default 50 is right for most sessions.

**Pagination.** Cursor-based, infinite-scroll or an explicit "Load older". Keep
`sources`/`from`/`to`/`days`/`limit` **identical** across pages — changing any of them
invalidates the cursor position, so changing a filter must reset to page 1. `nextCursor` is
`null` exactly when `hasMore` is `false`.

**Render `meta`, don't ignore it:**
- Show `meta.from`/`meta.to`/`meta.windowDays` as the effective window. The server clamps any
  span over 90 days, so what you asked for and what you got can differ.
- `meta.limitClamped: true` → a quiet note that the page size was reduced because many sources
  were selected. Selecting fewer sources allows a bigger page; that's a useful thing to tell
  the operator rather than hide.
- `meta.sources` is the canonical list actually queried — reconcile your filter chips against
  it after every response.

**Content-specific honesty, on the rows themselves:**
- **Chat.** A `chat_conversation_started` event proves a conversation happened and carries
  `metadata.conversationId` + `metadata.feature`. **It carries no messages.** Label the row
  "AI conversation started" with a copyable Dify conversation id — never "chat transcript",
  never an affordance that implies expanding it will show what was said. Chat content lives
  only in Dify; this is a pointer to look it up there.
- **`api_usage` rows** are route templates plus, on `/quota/begin`, a `feature`. Humanise
  the feature keys in copy (`chat_mentor` → "AI Mentor chat", `mains_eval` → "Mains
  evaluation") while keeping the raw key for logic. A `/quota/begin` row with
  `feature: null` predates the capture change — it means "unknown", not "no feature".
- **`identifierMasked`** on auth events is already masked server-side (`******7935`). Show it
  as-is; there is no unmasked value to reveal and no toggle to build.
- **Money rows** are already in major units.

**States**
- *Loading:* a few skeleton rows; subsequent pages append a loading row at the bottom.
- *Empty (filters applied):* "No <source> activity in this window" + a one-click "clear
  filters / widen to 90 days" escape. Never a dead end.
- *Empty (no filters):* distinguish the two real causes —
  (a) **nothing happened** — plausible for a dormant account; and
  (b) **nothing was captured** — the request trail starts at the interceptor's deploy date and
  is purged at 30 days, and the mobile-emitted events do not exist in any shipped build yet.
  If the window predates capture, say *"Request tracking started on <date>"* rather than
  implying the user was idle.
- *Error:* inline retry on the panel; keep already-loaded pages on screen.

---

## 4. Notes — `POST /sme/users/:id/notes`

Notes are written here and **read back through the timeline** (source `sme_audit_log`,
`action: 'SUPPORT_NOTE'`). There is no separate list endpoint — so the notes panel is a
composer plus a filtered view of the timeline.

**Composer:** `note` (required, 1–2000 chars — show a live counter), optional `category`
(≤60; offer the team's common tags as suggestions while keeping it free-text), optional
`author` (≤120).

⚠️ **`author` is self-reported and unverified.** The SME surface authenticates with a *shared*
`x-api-key` that carries no identity — the server cannot know who wrote a note. If the portal
has its own signed-in user, pre-fill `author` from it; otherwise leave it blank. Never present
`author` as an attested identity — no verified badge, no avatar that implies authentication.

**Behaviour:** a 201 returns the created note. Optimistic insert is fine, but **this write is
not best-effort** — unlike the rest of the SME audit trail, a DB failure surfaces as a 5xx here
on purpose, because the note *is* the request. On failure you must roll the optimistic row back
and keep the composer's text, or the operator loses what they typed.

Notes should also appear **inline in the main timeline**, interleaved chronologically with
everything else. That is the point of storing them in the audit log: "we called them at 15:10"
sits right next to the failures at 14:22. Give staff rows a visually distinct treatment so a
human action is never mistaken for a system event. The same panel shows other
`sme_audit_log` actions (`PREMIUM_GRANT`, `TRIAL_EXTEND`, `USER_NOTIFY`, `FEEDBACK_REPORT_*`…)
— those are staff actions too and belong in the same narrative.

**States**
- *Loading:* the notes list shares the timeline's loading state.
- *Empty:* "No notes yet" with the composer focused and ready — an invitation, not a void.
- *Submitting:* disable submit, keep the text. Never clear the field before the 201.
- *Error:* inline under the composer, with the text preserved and a retry.

---

## Cross-cutting rules

**Response shape.** Success bodies are **raw** — there is no `{ success, data }` envelope on
this API. Branch on HTTP status; error bodies (and only error bodies) carry
`{ success: false, message, error, statusCode, … }`. This is written out in full at the top of
`SME_ACTIVITY_TRAIL_API.md`, and it is the assumption that crashed a mobile release last week.
Do not port an envelope-unwrapping helper from another integration.

**Timezone.** Every day bucket, every `HH:MM` an operator reads aloud, every bare date sent to
the API: **IST (UTC+05:30)**. The server runs UTC; `at` and every `Date` field are UTC ISO.
Fields already labelled IST (`istDate`, `startedAtIST`) are pre-converted — don't convert twice.

**Linkability.** `/app-users/:id` must be shareable, and the timeline's filter + window state
belongs in the query string so an operator can paste "here's exactly what I was looking at"
into a ticket.

**Density over decoration.** This is internal tooling. Optimise for scannability and trust:
show denominators (`activity.windowDays`, `meta.from`/`meta.to`, `failureCount`), prefer
relative timestamps with absolute values on hover, and keep colour meaningful — reserve it for
`severity` and entitlement state rather than spending it on chrome.

**Copy-ability.** Operators live in ticket threads. The user id, support code, order ids,
Razorpay/Apple transaction ids and Dify conversation ids all need one-click copy.

**Three-state thinking, everywhere.** This page is full of fields where "false", "zero" and
"we don't know" are genuinely different answers: `permissionGranted` (`true`/`false`/`null`),
`quota.available`, `streak.available`, `feature` on `api_usage`, `lastRequestAt`. Every one of
them needs a distinct rendering. Collapsing unknown into zero is the single most likely way
this page misleads someone.

---

## What will be empty on day one — plan for it

Say this in the UI, not just in this doc. An operator who sees an empty panel and assumes
"nothing happened" will give a user the wrong answer.

| Panel / field | Why it may be empty | What to render |
|---|---|---|
| Purchase-funnel events (`paywall_viewed`, `upgrade_tapped`, `checkout_opened`, `checkout_abandoned`, `purchase_failed`) | Emitted by app code **not in any shipped build yet** | "Ships with the next app release" — not "no activity" |
| `client_error` rows / their incidents | Same | Same |
| `app_opened` / `app_backgrounded` | Same | Same |
| `pushTokens[].permissionGranted` | `null` until the next release; older tokens stay `null` forever | "Unknown (older app build)" — **not** "denied" |
| `chat_conversation_started` | Requires the app to send `conversationId` on a **successful** `/quota/finalize` for a **chat** feature — next release | "No AI conversations recorded" + the note that content lives in Dify regardless |
| `api_usage` older than ~30 days | Nightly purge; and nothing at all before the interceptor's deploy date | "Request tracking started on <date>" |
| `auth_events` older than 90 days | Nightly purge | State the retention rather than showing a blank tail |
| Quota for any day but today | Redis TTL at midnight IST — genuinely deleted | "Today only — quota history is not retained" |
| Streak | Neo4j has known stale/duplicate user nodes | The `unavailableReason`, never a zero |

Before shipping the page, run `SELECT min(created_at) FROM api_usage;` and use the real date in
that copy. Do **not** hardcode a guess, and do not promise a window the data doesn't have.

---

## Coverage contract — render EVERY data point (Definition of Done)

The page must **consume and visibly surface every field of every endpoint**. A field may live
in a tooltip, an expanded detail row, or a hover title — but it must be reachable. Do not
silently drop a field because it "didn't fit". Tick each box.

- [ ] **snapshot.user** → `id` · `supportCode` · `email` · `name` · `phoneNumber` · `phoneVerified` · `status` · `provider` · `createdAt`
- [ ] **snapshot.entitlement** → `isPremium` · `premiumState` · `premiumExpiresAt` · `subscriptionSource` · `trialEndsAt` · `trialDaysLeft`; `recentOrders[]`: `id` · `status` · `amount` · `currency` · `planType` · `paymentSource` · `premiumGrantedAt` · `createdAt`
- [ ] **snapshot.quota** → `available` · `scope` · `istDate` · `resetsAtISTMidnight` · `premium` · `features` (all keys, generically by `type`) · `unavailableReason`
- [ ] **snapshot.streak** → `available` · `currentStreak` · `maxStreak` · `lastActiveDate` · `unavailableReason`
- [ ] **snapshot.activity** → `windowDays` · `from` · `to` · `total` · every key of `counts` · `retention` (4 fields)
- [ ] **snapshot.lastSeen** → `lastLoginAt` · `lastActiveAt` · `lastSessionAt` · `lastRequestAt` · `lastRequestRoute` · `lastClientEventAt` · `lastClientEventType`
- [ ] **snapshot.clients** → `appVersions[]`: `appVersion` · `platform` · `lastSeenAt`; `distinctDeviceCount` · `deviceCountCapped`; `pushTokens[]`: `platform` · `isActive` · `permissionGranted` (3-state) · `lastUsedAt`
- [ ] **timeline.items[]** → `id` · `rowId` · `source` · `type` · `at` · `title` · `severity` · `data` (every key, per source — see the source table in the API doc)
- [ ] **timeline.meta** → `sources` · `limit` · `limitClamped` · `rowsFetched` · `rowCap` · `from` · `to` · `windowDays` · `retention`; plus `nextCursor` / `hasMore` driving pagination
- [ ] **incidents[]** → `id` · `oldestRowId` · `startedAt` · `endedAt` · `startedAtIST` · `istDate` · `failureCount` · `severity` · `title` · `sources` · `byLabel[]` · `byStatus[]` · `samples[]` (max 5)
- [ ] **incidents.meta** → `limit` · `gapMinutes` · `rowsScanned` · `rowCap` · `perSourceRowCap` · `rowCapHit` · `truncatedSources` · `from` · `to` · `windowDays` · `retention`; plus `nextCursor` / `hasMore`
- [ ] **note (201)** → `id` · `action` · `targetUserId` · `note` · `category` · `author` · `createdAt`
- [ ] **support-code lookup** → `id` · `supportCode` · `email` · `name` · `phoneNumber` · `status` · `isPremium` · `premiumState` · `createdAt`

The Swagger group **SME** at `/api/docs` is the live field list — diff against it before
calling the page complete.

---

## Hard reminders

- **Invoke `frontend-design` before building.** (Yes, again.)
- **Render every field — see the Coverage contract. No data point left unconsumed.**
- Success bodies are **raw**. Branch on status, not on `success`.
- `available: false` ≠ zero. `permissionGranted: null` ≠ denied. `deviceCountCapped` ≠ exactly 50.
- Group by **IST** days; never by the UTC date of `at`.
- Use the server's `title` strings; don't reimplement 17 formatters.
- The timeline proves a chat happened. It does not show what was said.
- Empty panels for mobile-emitted signals are **expected until the next app release** — say so
  on screen.
