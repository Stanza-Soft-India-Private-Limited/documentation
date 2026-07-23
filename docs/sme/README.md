# SME Portal — Integration Docs

Server-to-server API docs for the external **SME portal**. We expose the endpoints; the
SME team builds their own UI against them.

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Auth:** `x-api-key: <API_KEY_SECRET>` on every `/sme/*` request.
**Live Swagger:** `/api/docs` (group **SME**).

## Start here

**[WHAT_CHANGED_2026-07-23.md](./WHAT_CHANGED_2026-07-23.md)** — the cover note for the
current bundle: what is new, what corrected an earlier doc, the real production numbers each
endpoint returns today, and the `frontend-design` requirement that applies to **every** doc
below that results in a screen.

## Contents

| Doc | Covers |
|---|---|
| [WHAT_CHANGED_2026-07-23.md](./WHAT_CHANGED_2026-07-23.md) | Cover note — bundle contents, live production numbers per endpoint, contract corrections, null-vs-zero rules, and the mandatory `frontend-design` brief |
| [SME_PORTAL_API.md](./SME_PORTAL_API.md) | Master guide — users, transactions, blogs, notifications (summary), premium grant/revoke |
| [SME_TRIAL_EXTENSION_API.md](./SME_TRIAL_EXTENSION_API.md) | Trial extension in depth — extend/stack N days, revival of expired/churned users, notify pairing |
| [SME_FILTER_CONFIG_API.md](./SME_FILTER_CONFIG_API.md) | PYQ subject filter config — per-subject display name/icon/visibility/sort overrides, mains Optional↔GS flip, icon-key contract |
| [SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) | Push notifications in depth — per-user / broadcast / segment, deep-link `type` contract, premium-vs-free mechanics, caveats |
| [SME_FEEDBACK_API.md](./SME_FEEDBACK_API.md) | In-app feedback — report triage (status/replies), chat-feedback aggregates, chip-set / snooze config, survey authoring + lifecycle + results/CSV, deep-links, audit. **Corrected 2026-07-23: there is no response envelope.** Three distinct screens (triage queue · analytics panel · authoring tool); **mandates invoking the `frontend-design` skill** |
| [SME_APPLE_IAP_API.md](./SME_APPLE_IAP_API.md) | Apple IAP in depth — payments/orders via `source=APPLE`, the Apple event/notification stream on `/sme/webhooks`, subscription lifecycle, Razorpay-safe `/sme/webhooks` change |
| [SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) | DAU & usage analytics — live `/sme/analytics/*` endpoints (summary, dau two-series, features, per-user, retention, churn, premium-engagement, paywall, heatmap, endpoints, platforms, content, notifications) + the bounded `api_usage` capture/rollup |
| [SME_INSIGHTS_API.md](./SME_INSIGHTS_API.md) | Four decision endpoints — `/sme/content/question-quality` (statistically-flagged bad questions as a content **work queue**; `contradictsKey` = probable wrong answer key), `/sme/analytics/release-health` (per app-version × platform error rate + the force-update number; **NULL app_version = the pre-1.7 cohort**), `/sme/analytics/notification-effectiveness` (the **kill list**, ranked by wasted sends; **what `isRead` really means**), `/sme/analytics/onboarding-funnel` (signup → phone → onboarding → psychometric → first action, + the OTP/login failure signal). Each with a "How to use this data" section and the real production payload. **Mandates invoking the `frontend-design` skill** |
| [SME_ANALYTICS_FRONTEND_GUIDE.md](./SME_ANALYTICS_FRONTEND_GUIDE.md) | **Feed this to the portal's Claude/coding session.** Expressive per-endpoint response shapes + UX intent + suggested dashboard IA; **mandates invoking the `frontend-design` skill** before building any UI |
| [SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) | Per-user activity trail — `/sme/users/:id/{snapshot,timeline,incidents,notes}` + support-code lookup; the client-event sink; **raw responses, no envelope**; what the data can and cannot tell you; a "How to use this data" section per endpoint; **mandates invoking the `frontend-design` skill** |
| [SME_USER_DETAIL_PAGE_GUIDE.md](./SME_USER_DETAIL_PAGE_GUIDE.md) | **Feed this to the portal's Claude/coding session.** UI proposal to replace the `/app-users` sidebar drawer with a dedicated `/app-users/:id` page — IA, per-panel empty/loading/error states, coverage contract; **mandates invoking the `frontend-design` skill** |
| [CMS_ADMIN_API.md](./CMS_ADMIN_API.md) | `/cms/*` — PYQ / Mains / Psychometric CRUD + bulk (already live) |
| [CONTENT_DOC_SME_API.md](./CONTENT_DOC_SME_API.md) | `/content-doc-admin` — study documents from raw markdown (already live) |
| [rollout-and-verify.md](./rollout-and-verify.md) | Deploy / verify checklist for the `/sme/*` surface |

Also in `docs/` and relevant to the portal:
[../USER_DATA_EXPORT.md](../USER_DATA_EXPORT.md) — the per-user data export surface.

Related design spec (lives under the superpowers specs area, not here):
`../superpowers/specs/2026-06-16-sme-blog-reels-design.md` — blog editor built into the
reel management screen.

---

## Two rules that apply to every doc in this folder

1. **There is no global response envelope.** `ResponseInterceptor` exists in the codebase and
   is registered nowhere. Success bodies are RAW; errors are shaped by the exception filter.
   **Branch on the HTTP status code, never on the presence of `success`.** Some list
   endpoints have their own `{ data, total, page, limit, hasMore }` pagination wrapper —
   that is a per-endpoint shape, documented per endpoint, not a global rule.
2. **Invoke the `frontend-design` skill before writing any UI.** Every doc that results in a
   screen opens with the requirement and a paste-ready brief. It is a hard requirement.
