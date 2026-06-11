# Wylto Contact Sync — Design Spec

**Date:** 2026-06-11
**Status:** Approved (design), pending implementation
**Scope:** Backend only (NestJS). No app/web changes.

## Goal

Push PrepMonkey users into Wylto's CRM Contacts so the Wylto contact list is
populated with real names, emails, and a meaningful acquisition source — instead
of the current state where contacts are auto-created by the OTP WhatsApp send
with `name = email-username`, no email, and `Source = "Unknown"`.

The sync must cover **existing** users (the ~300 already phone-verified, who will
never re-trigger the signup/OTP flow) as well as **new** users going forward.

## Chosen trigger: `GET /api/v1/user/me`, gated by a sync flag

`/me` is JWT-guarded and is hit by **every logged-in user on every app open**
(splash/dashboard reads it for auth + onboarding status). It is the single
chokepoint that reaches existing users, not just new signups — which is why it
beats hooking the OTP-verify path (that only fires for brand-new verifications).

Crucially, `/me` today runs **zero DB queries** — the controller returns the
`user` object the JWT guard already loaded. The sync reads its gating fields off
that in-memory object and writes to the DB **only when a push is actually
needed**, so the hot path stays cheap.

## Wylto Contact API (confirmed against live spec `server.wylto.com/public/api-spec.json`)

| Operation | Method + Path | Lookup key | Notes |
|---|---|---|---|
| createContact | `POST /api/v1/contact` | phone in body | responses: 200/400/401/500 (no 404) |
| updateContactByPhoneNumber | `PUT /api/v1/contact?phoneNumber=<E164>` | `phoneNumber` query param | explicit **404** = "No contact matched the provided phone number" |

- **Auth:** `Authorization: Bearer <token>` (`bearerAuth` scheme — same scheme as
  the existing OTP send). Host is the same `server.wylto.com`.
- **Partial update semantics:** empty/omitted fields are ignored; only provided
  fields are updated. All body fields are optional at schema level.
- **Phone normalization:** Wylto auto-prepends `+` if missing; value must contain
  ≥10 digits.

### Fields we send

```
name        ← UserAuth.name        (from Google/Apple OAuth at signup)
email       ← UserAuth.email       (from OAuth)
phoneNumber ← UserAuth.phoneNumber (E.164, set at OTP verify)
source      = "prepmonkey-app"     (constant)
```

We deliberately do **not** set `status`, `group`, `tags` — leave Wylto defaults.
(Those resolve names → IDs via the app's configured lists and 400 if the name
doesn't exist, so we avoid them until there's a reason.)

## Components

### 1. Schema — two additive columns on `UserAuth`

Per project convention (add columns, never new tables; migrations run by hand —
do **not** assume `prisma migrate` ran):

```prisma
wyltoSyncedAt   DateTime?   // timestamp of last successful push
wyltoSyncHash   String?     // sha256("name|email|phoneNumber") at last push
```

The hash detects "synced data changed since last push" in one comparison without
storing each field's prior value. Migration SQL handed to the user to run
manually (see Implementation Notes).

### 2. `WyltoContactService` (new — `src/common/wylto/wylto-contact.service.ts`)

A small, isolated, testable client. Reuses the existing `HttpService`
(`@nestjs/axios`) and `WYLTO_API_KEY` bearer. Single public method:

```ts
async upsertContact(input: {
  name?: string;
  email?: string;
  phoneNumber: string;   // E.164
  source: string;
}): Promise<void>
```

Behaviour:
1. `PUT /api/v1/contact?phoneNumber=<E164>` with body `{ name, email, source }`
   — enriches the contact Wylto auto-created from the OTP WhatsApp send.
2. If the call throws an **AxiosError with status 404** → fall back to
   `POST /api/v1/contact` with body `{ name, email, phoneNumber, source }`.
3. Any other error (401/500/network) → **re-throw** so the caller can decide
   not to stamp the sync flag (it'll retry on the next `/me`). Log it.

Config: add `contactApiUrl` to `wylto.config.ts` (`server.wylto.com/api/v1/contact`,
overridable via `WYLTO_CONTACT_API_URL`). Token assumption — try the existing
`WYLTO_API_KEY`; only if Wylto returns 401 do we introduce `WYLTO_CONTACT_API_KEY`.

### 3. `UserService.syncToWyltoIfNeeded(user)` — fire-and-forget

Receives the **already-loaded** `user` object from the `/me` controller (no extra
read). Must **never throw** (wrapped in try/catch internally).

```
1. if (!user.phoneVerified || !user.phoneNumber) return;      // Wylto keys on phone; verification is skippable
2. hash = sha256(`${user.name ?? ''}|${user.email ?? ''}|${user.phoneNumber}`)
3. if (hash === user.wyltoSyncHash) return;                   // already synced, unchanged
4. await wyltoContactService.upsertContact({ name, email, phoneNumber, source: 'prepmonkey-app' })
5. on success → prisma.userAuth.update: { wyltoSyncedAt: now, wyltoSyncHash: hash }   // only DB write, only when needed
6. on error   → logger.warn(...); do NOT stamp → retries on next /me
```

> Dependency: the gating fields (`phoneNumber`, `phoneVerified`, `wyltoSyncedAt`,
> `wyltoSyncHash`) must be present on the object the JWT guard loads. If
> `JwtStrategy.validate` doesn't already select them, add them to that select —
> still no extra query, just more columns on the existing user load.

### 4. Controller wiring (`user.controller.ts`)

```ts
@Get('me')
getCurrentUser(@CurrentUser() user: any) {
  void this.userService.syncToWyltoIfNeeded(user).catch(() => {}); // fire-and-forget, no unhandled rejection
  return user;                                                     // unchanged, returns instantly
}
```

`UserModule` gains `HttpModule` (register with the same 30s timeout as auth) and
provides `WyltoContactService`.

## Data flow

```
app opens → GET /api/v1/user/me (JWT guard loads user)
          → controller returns user immediately
          → (async) syncToWyltoIfNeeded(user)
              ├─ no verified phone?         → skip
              ├─ hash == stored hash?       → skip
              └─ else → WyltoContactService.upsertContact
                          ├─ PUT ?phoneNumber=...      (200 → stamp flag)
                          ├─ 404 → POST /contact       (200 → stamp flag)
                          └─ 401/500/network → log, no stamp (retry next /me)
```

## Coverage

- **Existing ~300 verified users** — `wyltoSyncHash` is null → synced the first
  time they next open the app.
- **New users** — synced on the first `/me` after they verify phone.
- **Name/email edits later** — hash changes → exactly one re-sync.
- **Users who skipped phone verification** — never synced (no phone = no Wylto
  contact key). Acceptable and expected.

## Error handling principles

- `/me` never blocks on, slows for, or fails because of Wylto. Fire-and-forget.
- This is **not** a silent fallback to mock data (cf. no-silent-fallbacks rule):
  failures are logged at `warn` and the unstamped flag guarantees a retry. We are
  intentionally not blocking a hot read endpoint on a best-effort CRM side-channel.
- Wylto 404 on update is an expected control-flow branch (→ create), not an error.

## Out of scope / YAGNI

- No `status` / `group` / `tags` / `eventAt` mapping (no requirement yet).
- No bulk backfill job — `/me`-driven drip covers the existing base as users
  return, with no extra infra.
- No deletion sync (account delete → Wylto contact removal) — not requested.

## Implementation Notes

- Migration SQL (run manually, per project convention):
  ```sql
  ALTER TABLE "user_auth" ADD COLUMN "wylto_synced_at" TIMESTAMP(3);
  ALTER TABLE "user_auth" ADD COLUMN "wylto_sync_hash" TEXT;
  ```
  (Confirm the actual table/column naming — `@@map`/`@map` in `schema.prisma` —
  before running; Prisma model is `UserAuth`.)
- `sha256` via Node's built-in `crypto`.
- Reuse the OTP send's axios-via-`firstValueFrom(httpService.post(...))` pattern.

## Open items to confirm at implementation time

1. **Token scope** — does `WYLTO_API_KEY` carry contact-write permission? Try it;
   if 401, add `WYLTO_CONTACT_API_KEY`.
2. **JWT user load** — confirm `JwtStrategy.validate` selects (or is extended to
   select) the four gating fields.
