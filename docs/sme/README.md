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
| [SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) | Push notifications in depth — per-user / broadcast / segment, deep-link `type` contract, premium-vs-free mechanics, caveats |
| [SME_APPLE_IAP_API.md](./SME_APPLE_IAP_API.md) | Apple IAP in depth — payments/orders via `source=APPLE`, the Apple event/notification stream on `/sme/webhooks`, subscription lifecycle, Razorpay-safe `/sme/webhooks` change |
| [CMS_ADMIN_API.md](./CMS_ADMIN_API.md) | `/cms/*` — PYQ / Mains / Psychometric CRUD + bulk (already live) |
| [CONTENT_DOC_SME_API.md](./CONTENT_DOC_SME_API.md) | `/content-doc-admin` — study documents from raw markdown (already live) |
| [rollout-and-verify.md](./rollout-and-verify.md) | Deploy / verify checklist for the `/sme/*` surface |

Related design spec (lives under the superpowers specs area, not here):
`../superpowers/specs/2026-06-16-sme-blog-reels-design.md` — blog editor built into the
reel management screen.
