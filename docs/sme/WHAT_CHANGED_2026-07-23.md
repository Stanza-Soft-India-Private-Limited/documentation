# What's in this bundle — 2026-07-23

Three files. Two are new, one is a **correction to a doc you already have**.
Nothing else from `docs/sme/` is included, because nothing else changed.

---

## 🔴 1. `SME_FEEDBACK_API.md` — CORRECTION. Read this first.

**You already have a copy of this file** (it was in the 2026-07-22 bundle). **That copy is wrong.**

Line 8 of the old version said:

> All responses ride the global `{ success, message, data, timestamp, path }` envelope — read `data`.

**There is no such envelope.** `ResponseInterceptor` exists in the codebase but is registered
nowhere; the only global interceptor is `ApiUsageInterceptor`. **Success bodies come back RAW** —
a list endpoint returns a bare JSON array, an object endpoint returns a bare object.

Error bodies *are* shaped, but by the exception filter, not an envelope:
`{ success: false, message, error, statusCode, timestamp, path, method }`.

**Client rule: branch on the HTTP status code, never on the presence of `success`.**

This is not theoretical. Our own mobile app built envelope-decoding on that sentence, and the
surveys inbox crashed on-device — it received `[` where it expected `{`, and a survey card
silently rendered nothing for days. **If you have already written an unwrapping helper against
the old doc, that is the bug, and it will look like "the API returns nothing".**

Delete the old copy. Only the endpoint docs changed meaning — no endpoint behaviour changed.

---

## 🆕 2. `SME_ACTIVITY_TRAIL_API.md` — new API contract

Per-user activity trail: **"what happened to this user, end to end"** — the support/debug view.

- `GET /sme/users/:id/snapshot` — who they are, what they've paid, what's broken, right now
- `GET /sme/users/:id/timeline` — 17 merged sources, cursor-paginated
- `GET /sme/users/:id/incidents` — errors auto-clustered into "at 14:22, 3 failures on the payment route"
- `POST /sme/users/:id/notes` — support notes pinned to the user
- `GET /sme/users/by-support-code/:code` — resolve the short code shown in the app

**Read its "What this data can and cannot tell you" section before designing anything.** It is
not boilerplate — several fields are deliberately honest about missing data, and the UI has to
render those states rather than showing a zero. In particular:

- Request history starts when capture was deployed. There is no trail before that date — run the
  query in the doc to find the real earliest row rather than assuming a window.
- Chat **content** lives in Dify. We store the conversation **pointer**. The timeline proves a
  conversation happened and lets you look it up; it does not show what was said.
- Quota is today-only by design (it resets at IST midnight and yesterday is genuinely gone).
- Streak comes from a store we don't fully trust; it degrades to `available: false` with a
  reason. **Render that as "unavailable", never as 0.**

---

## 🆕 3. `SME_USER_DETAIL_PAGE_GUIDE.md` — new UI proposal

Replaces today's congested sidebar drawer on `/app-users` with a dedicated page.

> ### ⛔ If you are using an AI coding agent for this page, it MUST invoke the `frontend-design` skill FIRST
> The document opens with this instruction and it is a hard requirement, not a suggestion. Run
> `/frontend-design` **before writing any UI code** — before the first component, not after a
> draft exists. Every panel, table, timeline row and empty state goes through that skill.
> Do **not** let the agent reach for a default admin-dashboard template.
>
> Paste this along with the doc when you brief the agent:
>
> ```
> Invoke the /frontend-design skill first, before writing any UI code.
> Then build the SME user-detail page per SME_USER_DETAIL_PAGE_GUIDE.md,
> using SME_ACTIVITY_TRAIL_API.md as the authoritative field contract.
> Responses are RAW JSON — there is no {success,data} envelope.
> Render every "unavailable"/"no data yet" state explicitly; do not show 0.
> ```

---

## Status — what you can actually call today

**The backend is DEPLOYED and the endpoints are live** — verified 2026-07-23 (all
`/sme/users/:id/*` routes return a steady 401 without a key, and the migration columns are
confirmed present on production via `information_schema`).

You can re-confirm any time with a keyless request:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  https://app.stanzasoft.ai/api/v1/sme/users/00000000-0000-0000-0000-000000000000/snapshot
```

- **401** → deployed and guarded. You're good to go (add your `x-api-key`). ← current state
- **404** → not deployed. Design against the contract, but hold off on integration testing.

**Request history begins 2026-06-30** (measured on production, ~167k rows). Note it is a
**rolling 30-day window** — raw rows are purged nightly — so that start date moves forward
daily. Re-run `SELECT min(created_at) FROM api_usage;` rather than hardcoding it.

**Mobile-sourced signals are not live yet.** Client errors, the paywall/purchase funnel, app
session events and push-permission state only start flowing once the next app release ships
**and users update**. Those panels will be empty at first — that is expected. Please build them
to distinguish **"no data yet"** from **"nothing happened"**, because for the next few weeks
they mean very different things.
