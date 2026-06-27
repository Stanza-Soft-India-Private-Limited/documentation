# SME Portal — Management API Guide

End-to-end reference for the **SME portal** to manage the PrepMonkey app's users,
transactions, blogs, and notifications. This is the management/admin surface of the
app exposed server-to-server.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `BASE_URL` = `https://app.stanzasoft.ai`)
**Authentication:** `x-api-key: <API_KEY_SECRET>` header on **every** `/sme/*` request.
**Note:** All endpoints use the `/api/v1/` global prefix. Swagger UI at `/api/docs`
(group **SME**) documents these live with the `x-api-key` control.

```
GET /api/v1/sme/users
x-api-key: <API_KEY_SECRET>
```

- Missing/invalid key → **401** `{ "message": "Invalid or missing API key" }`.
- Money amounts in **responses** are in **rupees**; the `amount` you optionally pass
  to a premium grant is in **paise** (matches how the app stores prices).
- Pagination: list endpoints take `page` (default 1) + `limit` (default 20, max 100)
  and return `{ data, total, page, limit, hasMore }`.

---

## 1. User Management — `/sme/users`

Users originate from Cognito OAuth (Google/Apple) + onboarding, so there is **no
"create user"** endpoint. SME can list, view, edit (name/status), deactivate, delete,
grant/revoke premium, and notify.

### 1.1 List users
```
GET /sme/users?page=1&limit=20&search=&status=&premium=&onboarded=&platform=
```
| Query | Type | Description |
|---|---|---|
| page | number | default 1 |
| limit | number | default 20, max 100 |
| search | string | matches email / username / name / phone (case-insensitive) |
| status | enum | ACTIVE, INACTIVE, SUSPENDED, LOCKED, SUBSCRIBED, UNSUBSCRIBED |
| premium | boolean | `true` = currently premium (SUBSCRIBED and not expired) |
| onboarded | boolean | filter by onboarding completion |
| platform | enum | `ios` \| `android` \| `web` (has an active device token) |

**Response 200:**
```json
{
  "data": [
    {
      "id": "0d2f…",
      "email": "user@example.com",
      "username": "user",
      "name": "Asha R",
      "phoneNumber": "+919876543210",
      "phoneVerified": true,
      "provider": "google",
      "role": "USER",
      "status": "SUBSCRIBED",
      "isPremium": true,
      "premiumState": "Premium",
      "premiumExpiresAt": "2026-12-01T00:00:00.000Z",
      "subscriptionSource": "RAZORPAY",
      "onboardingCompleted": true,
      "lastLoginAt": "2026-06-15T09:12:00.000Z",
      "createdAt": "2026-01-10T00:00:00.000Z"
    }
  ],
  "total": 969,
  "page": 1,
  "limit": 20,
  "hasMore": true
}
```
`premiumState` is a derived funnel label: `Premium` / `Trial` / `Trial Ended` /
`Churned` / `Downloaded` (computed from live subscription state, not the raw column).

### 1.2 Get a user
```
GET /sme/users/:id
```
**Response 200:** all summary fields plus:
```json
{
  "...": "summary fields above",
  "profile": { "aspirantType": "FULL_TIME", "attempt_year": 2027, "onboardingCompleted": true, "...": "full UserProfile" },
  "platforms": ["android", "web"],
  "recentOrders": [
    {
      "id": "ord_…", "amount": 599, "currency": "INR", "status": "PAID",
      "planType": "MONTHLY", "paymentSource": "RAZORPAY",
      "premiumGrantedAt": "2026-06-01T…", "createdAt": "2026-06-01T…",
      "latestPayment": { "id": "pay_…", "status": "CAPTURED", "method": "upi", "amount": 599 }
    }
  ]
}
```
`404` if the user doesn't exist.

### 1.3 Update a user (name / status only)
```
PATCH /sme/users/:id
{ "name": "New Name", "status": "ACTIVE" }
```
Both fields optional. `status` ∈ ACTIVE, INACTIVE, SUSPENDED, LOCKED, SUBSCRIBED,
UNSUBSCRIBED. Phone, email and role are read-only. Returns the updated user summary.

### 1.4 Deactivate (soft delete — reversible)
```
POST /sme/users/:id/deactivate
```
Sets `status=SUSPENDED` and revokes all active sessions. **Response:**
`{ "success": true, "status": "SUSPENDED" }`. Reverse by `PATCH`-ing status back to ACTIVE.

### 1.5 Delete (hard delete — irreversible)
```
DELETE /sme/users/:id
```
Permanently purges the user: **Postgres cascade** (profile, orders, payments, content,
attempts, device tokens, notifications…) **+ Neo4j** node and relationships. **Cannot be
undone.** Response: `{ "success": true, "message": "Account deleted successfully" }`.

### 1.6 Grant premium
```
POST /sme/users/:id/premium
{ "plan": "MONTHLY", "amount": 59900, "reference": "rzp_pay_XXXX" }
```
| Field | Required | Description |
|---|---|---|
| plan | yes | `MONTHLY` (30d) or `ANNUAL` (365d) |
| amount | no | paise to record on the transaction; defaults to the configured plan price |
| reference | no | external reference (e.g. the real Razorpay payment id being reconciled), stored on the order/payment notes + audit log |

Creates an **auditable** `Order(source=MANUAL, status=PAID)` + `Payment(CAPTURED)`,
sets `status=SUBSCRIBED` + `subscriptionSource=MANUAL`, **extends** any existing
non-expired premium (otherwise starts now), syncs Neo4j tier + Wylto CRM. Use this to
**fix a payment that was taken but premium never set** (or to comp a user).

**Response 200:**
```json
{
  "success": true, "plan": "MONTHLY",
  "premiumExpiresAt": "2026-07-16T…",
  "orderId": "ord_…", "paymentId": "pay_…", "amount": 599
}
```

### 1.7 Revoke premium
```
DELETE /sme/users/:id/premium
```
Force-expires premium now: `status=UNSUBSCRIBED`, `premiumExpiresAt=now`, Neo4j tier→free,
Wylto→Churned. Response: `{ "success": true, "expiredAt": "2026-06-16T…" }`.

### 1.8 Notify a single user
```
POST /sme/users/:id/notify
{ "title": "…", "body": "…", "type": "home", "id": "optional-entity-id" }
```
Sends an FCM push to the user's active devices and writes it to their in-app feed.
`type` is the deep-link route (`home`, `chat`, `daily_task`, `mains_question`, …;
default `home`). **Response:** `{ "successCount": 1, "failureCount": 0, "deactivatedTokens": 0 }`.

---

## 2. Transactions (view-only) — `/sme/transactions`, `/sme/orders`, `/sme/webhooks`

Read-only. Razorpay **and** Apple data. Amounts in rupees.

### 2.1 List payments
```
GET /sme/transactions?page=1&limit=20&userId=&paymentStatus=&source=&startDate=&endDate=&search=
```
| Query | Description |
|---|---|
| paymentStatus | PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED |
| source | RAZORPAY, APPLE, MANUAL (filters by the parent order's source) |
| userId | payments for one user |
| startDate / endDate | ISO date bounds on `createdAt` |
| search | razorpay payment id / payer email / user email |

**Response 200** `data[]`:
```json
{
  "id": "pay_…", "orderId": "ord_…", "razorpayPaymentId": "pay_Nxxx",
  "amount": 599, "currency": "INR", "status": "CAPTURED", "method": "upi",
  "email": "user@example.com", "capturedAt": "…", "createdAt": "…",
  "order": { "id": "ord_…", "userId": "…", "planType": "MONTHLY", "paymentSource": "RAZORPAY",
             "user": { "id": "…", "email": "…", "name": "…", "phoneNumber": "…" } }
}
```

### 2.2 Payment detail
```
GET /sme/transactions/:id
```
Same shape + `notes`. `404` if not found.

### 2.3 List orders / order detail
```
GET /sme/orders?page=&limit=&userId=&orderStatus=&source=&startDate=&endDate=&search=
GET /sme/orders/:id
```
`orderStatus` ∈ CREATED, PAID, FAILED, CANCELLED. Each order includes its `payments[]`,
`planType`, `paymentSource`, and **`premiumGrantedAt`**. **Reconcile workflow:** an order
with `status=PAID` but `premiumGrantedAt=null` is a payment that never granted premium →
grant it via §1.6 (the Razorpay webhook now also auto-grants, so new ones self-heal).

### 2.4 Webhook events
```
GET /sme/webhooks?page=&limit=&startDate=&endDate=&search=
```
Raw Razorpay/Apple webhook events for debugging stuck payments.

---

## 3. Blog Management — `/sme/blogs`

Works **in tandem with reels** (1 blog per reel). **Build the blog editor into the reel
management screen, not a separate screen** — see
`../superpowers/specs/2026-06-16-sme-blog-reels-design.md`. Markdown format = the
content/image/quiz syntax from `CONTENT_DOC_SME_API.md` (quiz blocks are stripped for blogs).

| Method | Path | Body | Notes |
|---|---|---|---|
| GET | `/sme/blogs?page=&limit=&hasBlog=&search=` | — | reels + `hasBlog` + blog summary |
| GET | `/sme/blogs/:reelId` | — | full blog (title, rawMarkdown, sections) |
| POST | `/sme/blogs` | `{ reelId, title, rawMarkdown }` | create; parses sections |
| PATCH | `/sme/blogs/:reelId` | `{ title?, rawMarkdown? }` | re-parses if rawMarkdown sent |
| DELETE | `/sme/blogs/:reelId` | — | deletes the blog; reel untouched |

**List response 200** `data[]`:
```json
{ "reelId": "…", "title": "…", "subject": "Polity", "status": "READY",
  "date": "…", "hasBlog": true,
  "blog": { "id": "…", "title": "…", "createdAt": "…", "updatedAt": "…" } }
```
**Create/Get response:** the `ReelBlog` row incl. `sections` (array of
`{ order, type: "content", markdown }` / `{ order, type: "image", url, alt }`).
`404` on get/update/delete if the reel has no blog.

---

## 4. Notifications — `/sme/notifications`

> **Full guide: [`SME_NOTIFICATIONS_API.md`](./SME_NOTIFICATIONS_API.md)** — covers per-user
> vs broadcast vs segment, the deep-link `type` contract, how premium/free is decided
> (topic subscription vs segment query), and all delivery caveats. The summary below is
> the quick reference.

Single-user notify lives at §1.8. These are the audience sends.

### 4.1 Broadcast to all
```
POST /sme/notifications/broadcast
{ "title": "…", "body": "…", "type": "home", "id": "optional" }
```
Sends to the `all_users` FCM topic. **Response:** `{ "messageId": "…" }`.

### 4.2 Segment send
```
POST /sme/notifications/segment
{ "segment": "premium", "title": "…", "body": "…", "type": "home", "id": "optional" }
```
`segment` ∈ `premium` (active paid) / `trial` (active free trial) / `free` (everyone else).
Resolves matching users and fans out a per-user push. **Response:**
`{ "segment": "premium", "matchedUsers": 412, "successCount": 380, "failureCount": 32 }`.

---

## 5. Existing content handoff — ✅ ALREADY IMPLEMENTED & LIVE (do NOT rebuild)

> **For the SME portal agent:** everything in this section is **already shipped and in
> production**. It is listed here only so you have the complete picture. **Do not
> re-implement or re-scope it** — the SME portal already integrates these. The NEW work
> in this doc is §1–§4 (`/sme/*`). The only NEW content-adjacent work is **blog
> management (§3)**, which builds on the existing reels endpoints below.

These are `@Public()` with **no** `x-api-key` (unlike the `/sme/*` routes above). Full guides:

- **`CMS_ADMIN_API.md`** — `/cms/pyq`, `/cms/mains`, `/cms/psychometric` (PYQ / Mains /
  Psychometric CRUD + bulk, paginated lists). **DONE.**
- **`CONTENT_DOC_SME_API.md`** — `/content-doc-admin` (study documents from raw markdown
  with embedded quiz blocks → parsed sections). **DONE.**
- **Reels / video** — `GET /reels` (list, filters), `POST /videos/upload-url` (Mux upload),
  `PUT /reels/bulk` (metadata after processing), `DELETE /reels/:videoId`. **DONE.** Blogs
  for these reels are the only NEW piece — managed via §3.

**Blog render note (verified from client code):** blog `sections` produced by §3 render
correctly on both mobile and web with **zero** extra formatting — the apps consume the
stored `sections` verbatim (`{order, type:'content', markdown}` / `{order, type:'image',
url, alt}`, lowercase discriminators) via `GET /reels/:reelId/blog`. The SME blog
endpoints reuse the same parser as the existing live blog admin, so output is identical.

---

## Common errors

| Status | Cause | Fix |
|---|---|---|
| 401 | Missing/invalid `x-api-key` | Send the `x-api-key` header = `API_KEY_SECRET` |
| 400 | Validation (bad enum, missing required field) | Check the body against the tables above |
| 404 | Unknown user / payment / order / blog | Verify the id |

## Audit trail

Every privileged/destructive SME action (update, deactivate, hard-delete, premium
grant/revoke, notify, blog create/update/delete) is recorded in `sme_audit_log`
(action, target user, before/after snapshot, optional reference, timestamp).
