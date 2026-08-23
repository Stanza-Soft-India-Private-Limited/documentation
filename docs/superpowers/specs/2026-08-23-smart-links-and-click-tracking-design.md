# Smart links (`go.prepmonkey.com`) + first-class click tracking — design

**Date:** 2026-08-23
**Repos:** `prepmonkey-web`, `backend`, `upsc_app`
**Status:** approved in chat, not yet implemented

---

## 1. The two problems

**A. Links tapped inside another app's browser never reach the app.** Instagram, WhatsApp
and most email clients render a tapped link in their own embedded WebView. The OS is never
asked to resolve the URL, so iOS Universal Links and Android App Links cannot fire. On
Android this was solved on 2026-08-17 with an `intent://` fallback. On iOS it is still
broken, and it is why a campaign link distributed through Instagram reached almost nobody.

**B. There is no click data at all.** The funnel's first stage is `viewed`, stamped
server-side when the paywall *resolves*. `/open/*` is a static page on Vercel that never
touches the backend, so a tap that does not end in a resolved paywall leaves no trace
anywhere. "Did anyone click?" is currently unanswerable except by inference from orders.

## 2. Non-goals

- Deferred deep linking (install → first launch lands on the original target). Explicitly
  out of scope; the decision to send post-install users to Home stands.
- Any third-party attribution SDK (Branch, AppsFlyer, Adjust). Previously evaluated and
  rejected on cost.
- Retroactive data. Capture begins at deploy, exactly as the `viewed` stage did.
- Changing the behaviour of `app.prepmonkey.com/open/*` for links that already work.

## 3. The rule this whole design exists to satisfy

> **iOS will not open the app from a Universal Link if the page you are currently on is
> served from the same domain as that link.**

This is documented Apple behaviour, not a bug. It is fatal here: the `/open/*` fallback page
and the Universal Link target are both `app.prepmonkey.com`, so a page served from
`app.prepmonkey.com` can never hand off to the app. The only remaining lever would be the
custom scheme `com.stanzasoft.upscbuddy://`, which raises a system *"address is invalid"*
alert when the app is absent — shown to precisely the cohort we want to convert to installs.
A user gesture does **not** suppress that alert.

A second host removes the constraint. `go.prepmonkey.com` was added to the
`prepmonkey_web_app` Vercel project and verified on 2026-08-23:

| check | result |
|---|---|
| `CNAME go` → `f5b7c85a7ff028fd.vercel-dns-017.com.` | ✅ Cloudflare, **DNS only** (grey cloud) |
| TLS | ✅ `ssl_verify_result=0` |
| `GET https://go.prepmonkey.com/` | ✅ `200`, 0 redirects |
| `GET /.well-known/apple-app-site-association` | ✅ `200`, `application/json`, **0 redirects** |

Cloudflare proxying must stay OFF for this host. Apple rejects an AASA served through any
redirect, and the failure is silent — links simply stop opening the app.

## 4. Routing

`app.prepmonkey.com/open/*` remains the canonical Universal Link / App Link target and its
behaviour for links that already work is unchanged.

| Source | Host tapped | Behaviour |
|---|---|---|
| Messages, Safari, share sheet | `app.*` | OS opens app directly — **unchanged** |
| Messages, Safari, share sheet | `go.*` | OS opens app directly *(after 1.9 entitlement; until then → interstitial → app)* |
| Instagram / WhatsApp / in-app browser, iOS | `go.*` | interstitial → cross-host handoff → app, or App Store |
| Instagram / WhatsApp / in-app browser, iOS | `app.*` | **bounce to `go.*`** → interstitial → app *(rescues links already published)* |
| Any in-app browser, Android | either | `intent://` → app, or `browser_fallback_url` → Play |
| Desktop | either | equivalent web screen (existing `resolveOpenTargetToWebPath`) |

### 4.1 The bounce, and the loop it must not create

If `app.*/open/*` renders at all on a phone, the OS was never asked or declined — the app did
not open. So on iOS that page may safely redirect to `go.*/open/*`.

🔴 **Loop hazard.** The interstitial's handoff button points back at `app.*/open/*`. If the
app is absent, that URL loads as a web page and would bounce to `go.*` again, forever.

**Mitigation:** the handoff URL carries `?nb=1` (no-bounce). `app.*/open/*` bounces only when
`nb` is absent. With `nb=1` present it renders today's behaviour (store / desktop web). This
flag is load-bearing; it must be covered by a test.

## 5. Data model

### 5.1 Why a new table, having seriously considered not adding one

The house rule is to prefer new columns on existing tables. `auth_events` was evaluated as
the host — it is already a generalised client-event sink (`POST /diagnostics/events`), it is
`@Public()`, rate-limited, tolerant, and has `userId`/`platform`/`metadata`. It was rejected
for four concrete reasons:

1. **`metadata` is `@db.Json`, not `Jsonb`.** `clickId` would live inside it and could not be
   indexed, making both the stitch lookup and every funnel query a sequential scan.
2. **The IP cap is 300/hr and Indian mobile carriers use CGNAT.** Thousands of Jio/Airtel
   users share a public IP, so a marketing blast — the exact event we are trying to measure —
   would silently trip the cap and drop real clicks. That is the "patchy" outcome this work
   exists to eliminate.
3. **The sink is lossy by design** (`insert dropped`, silent 429). Acceptable for diagnostics,
   not for a number anyone will make spend decisions from.
4. **A 90-day purge is planned for it.** Campaign analysis outlives that.

A purpose-built table also keeps marketing analytics out of a diagnostics table whose stated
contract is best-effort.

⚠️ **Correction to reasons 2 and 3 above:** the IP cap and the lossy drops are properties of the
diagnostics *endpoint*, not of `auth_events` itself — our own service sets its own limits, so they
would not have applied. Only retention and volume genuinely justify the separate table. Recorded so
the decision is re-litigated on the real reasons if it ever is.

### 5.2 `link_events`

| column | type | notes |
|---|---|---|
| `id` | uuid pk | |
| `click_id` | uuid, indexed | one per interstitial render; ties the chain together |
| `visitor_id` | string?, indexed | persisted per browser → "how many times did this person click" |
| `kind` | enum | `CLICK` · `APP_OPEN` · `JOIN_TAP` |
| `target_type` | string | `offer`, `reel`, `doc`, `pyq`, … — generic per the agreed scope |
| `target_id` | string? | campaign code, reel id, … |
| `user_id` | string?, indexed | null at click; backfilled on sign-in |
| `platform` | string? | `ios` / `android` / `desktop` |
| `source` | string? | from `?src=` (e.g. `instagram_bio`) |
| `referrer` | string? | truncated |
| `ip_hash` | varchar(64)? | SHA-256. Raw IP is never stored — mirrors `auth_events` |
| `user_agent` | string? | truncated |
| `is_bot` | boolean | UA-classified at ingest; excluded from every count by default |
| `created_at` | timestamptz | |

Indexes: `(click_id)`, `(user_id, created_at)`, `(target_type, target_id, created_at)`,
`(kind, created_at)`, `(visitor_id, created_at)`.

Uniqueness: `(click_id, kind)` — a reload or a retried beacon cannot double-count.

### 5.3 Identity

`click_id` is minted per interstitial render. `visitor_id` is a first-party cookie on
`go.prepmonkey.com`, stable across visits. Neither is PII and neither is ever shown to a user.

The stitch: the handoff URL carries `?cid=<click_id>`. The app already parses a deep link's
query string into `params` inside `DeepLinkMapper.fromUrl`, so the id rides inward with no
change to the URL grammar. The app holds it and calls `POST /links/claim` once authenticated;
the backend backfills `user_id` onto every row sharing that `click_id`/`visitor_id`. Clicks
from people who never sign in stay in a visible **unattributed** bucket rather than vanishing.

## 6. Backend surface

All three are `@Public()`, rate-limited, best-effort, and respond raw — no envelope
(`ResponseInterceptor` is not registered globally).

| endpoint | who calls it | purpose |
|---|---|---|
| `POST /links/click` | interstitial, **server-side during render** | logs `CLICK` |
| `POST /links/event` | app | logs `APP_OPEN`, `JOIN_TAP` |
| `POST /links/claim` | app + web, authenticated | binds `click_id`/`visitor_id` → `user_id` |

**Logging happens server-side during SSR, not from browser JavaScript.** It cannot be blocked
by an ad blocker, needs no consent banner for a first-party count, and works with JS disabled.
The cost is that bots and link-preview fetchers inflate counts, which `is_bot` handles by
classifying at ingest and excluding by default.

Rate limiting must be keyed on `visitor_id` first and fall back to `ip_hash`, with a ceiling
far above the diagnostics sink's, for the CGNAT reason in §5.1. Limits fail **open**.

### 6.1 SME funnel

`clicked` is added as the stage in front of `viewed`. `JOIN_TAP` becomes a real client-emitted
event on all three platforms, which retires the inferred iOS `TAP` signal
(`OFFER_ABANDON_SIGNALS`) — that heuristic exists only because Apple writes no order until a
transaction completes, and a first-class event is strictly better.

## 7. App changes (ride 1.9)

1. `applinks:go.prepmonkey.com` added to `iosApp.entitlements`; matching `go.prepmonkey.com`
   host on the existing `/open` intent-filter in `AndroidManifest.xml`. Upgrades `go.*` links
   from one tap to zero from sources where the OS *is* asked.
2. Capture `cid` in `DeepLinkMapper.fromUrl` into a `LinkAttribution` singleton.
3. Emit `APP_OPEN` on deep-link consumption; emit `JOIN_TAP` from the campaign CTA.
4. Call `POST /links/claim` after authentication.

⚠️ Kotlin block comments nest — never write a bare `/*` sequence (e.g. `/open/*`) inside a
KDoc in these files; it breaks the file.

## 8. Phasing

**Phase 1 — web + backend, ships without an app release and works on the 1.8 app already
installed.** Interstitial, bounce, `link_events`, the three endpoints, funnel stage.
This is the phase that fixes Instagram.

**Phase 2 — app, rides 1.9.** Entitlement/manifest hosts, `cid` capture, `APP_OPEN`,
`JOIN_TAP`, `claim`.

Phase 1 does not depend on Phase 2. Phase 2 improves tap count and completes attribution.

## 9. Loose end this surfaces

🔴 `go.prepmonkey.com` currently serves the **entire web app**, including login — a second
public front door nobody intended, and duplicate content. Phase 1 must restrict `go.*` to
`/open/*` and redirect everything else to `app.*`.

## 10. Verification

Automated:
- `nb=1` prevents the bounce loop (unit test on the routing helper).
- `(click_id, kind)` uniqueness rejects a double-fire.
- Bot UAs classified and excluded from counts.
- `resolveOpenTargetToWebPath` unchanged for every existing target (regression).

Manual, on a real device — the mechanism cannot be proven any other way:
1. iPhone, app installed, tap a `go.*` link **from an Instagram bio** → interstitial → one tap
   → app opens on target.
2. Same, app deleted → interstitial → App Store. **No "address is invalid" alert at any point.**
3. iPhone, tap an old `app.*` link from Instagram → bounces to `go.*` → app opens.
4. Confirm no redirect loop when the app is absent (case 2 exercises `nb=1`).
5. Android, both hosts, from Instagram → app opens via `intent://`.
6. Desktop, both hosts → correct web screen.
7. `link_event` rows land with the right `kind`, and `user_id` backfills after sign-in.

⚠️ Step 1 relies on Instagram's WebView honouring a cross-host Universal Link on tap. This is
the one assumption in the design that has not been verified on hardware. If it fails, the
fallback is the custom scheme with its error-alert trade-off, and that changes the UX decision
— so run step 1 before building anything on top of Phase 1.
