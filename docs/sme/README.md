# SME Portal — Integration Docs

Server-to-server API docs for the external **SME portal**. We expose the endpoints; the
SME team builds their own UI against them.

**Base URL:** `https://app.stanzasoft.ai/api/v1`
**Auth:** `x-api-key: <API_KEY_SECRET>` on every `/sme/*` request.
**Live Swagger:** `/api/docs` (group **SME**).

## Contents

| Doc | Covers |
|---|---|
| [SME_PORTAL_API.md](./SME_PORTAL_API.md) | Master guide — users, transactions, blogs, notifications (summary), premium grant/revoke |
| [SME_TRIAL_EXTENSION_API.md](./SME_TRIAL_EXTENSION_API.md) | Trial extension in depth — extend/stack N days, revival of expired/churned users, notify pairing |
| [SME_FILTER_CONFIG_API.md](./SME_FILTER_CONFIG_API.md) | PYQ subject filter config — per-subject display name/icon/visibility/sort overrides, mains Optional↔GS flip, icon-key contract |
| [SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) | Push notifications in depth — per-user / broadcast / segment, deep-link `type` contract, premium-vs-free mechanics, caveats |
| [SME_FEEDBACK_API.md](./SME_FEEDBACK_API.md) | In-app feedback — report triage (status/replies), chat-feedback aggregates, chip-set / snooze config, survey authoring + lifecycle + results/CSV, deep-links, audit |
| [SME_APPLE_IAP_API.md](./SME_APPLE_IAP_API.md) | Apple IAP in depth — payments/orders via `source=APPLE`, the Apple event/notification stream on `/sme/webhooks`, subscription lifecycle, Razorpay-safe `/sme/webhooks` change |
| [SME_USAGE_ANALYTICS.md](./SME_USAGE_ANALYTICS.md) | DAU & usage analytics — live `/sme/analytics/*` endpoints (summary, dau two-series, features, per-user, retention, churn, premium-engagement, paywall, heatmap, endpoints, platforms, content, notifications) + the bounded `api_usage` capture/rollup |
| [SME_INSIGHTS_API.md](./SME_INSIGHTS_API.md) | Decision endpoints — `/sme/content/question-quality` (statistically-flagged bad questions, ranked as a work queue), `/sme/analytics/release-health` (per app-version × platform error rate + force-update signal; **NULL app_version = the pre-1.7 cohort**), `/sme/analytics/notification-effectiveness` (read-rate by type + worst-performer kill-list; **what `isRead` really means**) |
| [SME_ANALYTICS_FRONTEND_GUIDE.md](./SME_ANALYTICS_FRONTEND_GUIDE.md) | **Feed this to the portal's Claude/coding session.** Expressive per-endpoint response shapes + UX intent + suggested dashboard IA; **mandates invoking the `frontend-design` skill** before building any UI |
| [SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) | Per-user activity trail — `/sme/users/:id/{snapshot,timeline,incidents,notes}` + support-code lookup; the client-event sink; **raw responses, no envelope**; what the data can and cannot tell you |
| [SME_USER_DETAIL_PAGE_GUIDE.md](./SME_USER_DETAIL_PAGE_GUIDE.md) | **Feed this to the portal's Claude/coding session.** UI proposal to replace the `/app-users` sidebar drawer with a dedicated `/app-users/:id` page — IA, per-panel empty/loading/error states, coverage contract; **mandates invoking the `frontend-design` skill** |
| [CMS_ADMIN_API.md](./CMS_ADMIN_API.md) | `/cms/*` — PYQ / Mains / Psychometric CRUD + bulk (already live) |
| [CONTENT_DOC_SME_API.md](./CONTENT_DOC_SME_API.md) | `/content-doc-admin` — study documents from raw markdown (already live) |
| [rollout-and-verify.md](./rollout-and-verify.md) | Deploy / verify checklist for the `/sme/*` surface |

Related design spec (lives under the superpowers specs area, not here):
`../superpowers/specs/2026-06-16-sme-blog-reels-design.md` — blog editor built into the
reel management screen.
