# Staging environment — `staging-api.prepmonkey.com`

An isolated second stack for QA (app + web), rehearsing risky features, and proving migrations
before they touch production.

**Data is content-only.** No real user's phone number or email is copied. That is a privacy
decision, and it is also the thing that makes the notification sandbox safe: with no rows in
`device_tokens`, the notification engine physically cannot reach a real user's phone.

---

## Inventory

| Piece | Status |
|---|---|
| Staging RDS Postgres | ✅ `prepmonkeytiote.…ap-south-1.rds.amazonaws.com` |
| Valkey (node-based) | ✅ `prepmonkey-staging.g4d90o.ng.0001.aps1.cache.amazonaws.com:6379` |
| Neo4j (Aura) | ✅ `21d04b5f` (db `21d04b5f`) — **isolation confirmed with evidence**, see below |
| Razorpay test plans | ✅ 4 plans, all test-mode |
| `.env.staging` | ✅ complete |
| Mobile app | ✅ `ApiConfig.USE_STAGING = true` drives host + Razorpay key |
| Mau service | ✅ deployed — `prepmonkeytesting-alb-449120516.ap-south-1.elb.amazonaws.com` |
| Migrations | ✅ replayed from zero, **"No difference detected"** |
| Content clone | ✅ 17 tables, row counts identical to production |
| `exam_plans` | ✅ seeded, test-mode ids only |
| DNS + TLS | ✅ `https://staging-api.prepmonkey.com` — Mau issued the cert, Cloudflare CNAME (grey cloud) to the ALB |
| Razorpay webhook | ✅ registered at `/api/v1/webhooks/razorpay`, signature verified live |

Production is `upscpostgres.*`; staging is `prepmonkeytiote.*`. **Every script here prints both
hosts and refuses to run if they match.**

---

# PART A — provisioning

## A1. ElastiCache Valkey  🔴 required

An unreachable Redis makes the app **hang forever at boot** rather than crash (four
`onModuleInit` hooks `await queue.add()` with `maxRetriesPerRequest: null`). "Starts but never
listens" has exactly one cause.

1. AWS Console → **ElastiCache → Parameter groups → Create parameter group**
   - Family `valkey8`, name `prepmonkey-staging-noeviction`
   - Edit parameters → find `maxmemory-policy` → set to **`noeviction`** → Save

   ✅ **Expected:** the group lists `maxmemory-policy = noeviction`.

   > **Do not skip this.** BullMQ requires `noeviction`; the ElastiCache default evicts keys and
   > your queues will silently lose jobs. Production hit this exact problem — the fix was
   > documented in `README.md` and has since been deleted from it.

2. **ElastiCache → Valkey caches → Create**, smallest node, single node, no replica,
   **same VPC and subnets as the staging RDS**, parameter group = the one from step 1.
3. Security group: allow inbound **6379** from the Mau/Fargate service's security group.
4. Copy the **primary endpoint** into `REDIS_URL` in `.env.staging` as `redis://<endpoint>:6379`.

   ✅ **Expected:** the value does **not** contain `upsc-valkey`. That string is production.

   > `src/config/redis.config.ts:31-33` hardcodes the **production** ElastiCache hostname as the
   > default under `NODE_ENV=production`. A missing or typo'd `REDIS_URL` therefore silently
   > attaches staging to prod's cache, sessions, rate limiters, quota counters and queues.

## A2. Neo4j — Aura Free instance

Production already runs on Aura (`AURA_INSTANCEID` is in `.env.production`), so this matches the
existing pattern.

1. <https://console.neo4j.io> → **New Instance → AuraDB Free**, region `asia-south1` or nearest.
2. **Download the credentials file when prompted — the password is shown once.**
3. Fill `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE`.
4. ⚠️ **AuraDB Free auto-PAUSES after ~3 idle days.** Symptom: `/health/full` says `neo4j: unhealthy` and
   `/pyq/:id/details`, `/pyq/metrics`, bookmarks 500. Fix: console → instance → Resume (shows RESUMING), **then
   redeploy/restart the staging API** — a process that booted while Aura was paused keeps reporting unhealthy even
   after Aura is back (seen 2026-10-06: direct driver connect OK, API still unhealthy until `mau deploy`).
   Probe 15× before trusting it. To get a bearer for API probes without a device: `scripts/staging-token.ts`.

   ⚠️ **The username and the database are BOTH the instance id, not `neo4j`.** Verified
   live on this instance (`21d04b5f`): `SHOW DATABASES` lists only `21d04b5f` and
   `system`; connecting as user `neo4j` fails with
   `Neo.ClientError.Security.Unauthorized`, and database `neo4j` fails with *"does not
   exist"*. Use whatever the downloaded credentials file says and **do not "correct" it
   to `neo4j`** — that assumption cost one redeploy.

   ✅ **Expected:** instance status **Running**, the URI differs from prod's, and a
   connectivity check returns a node count of **0** (proving it is fresh and isolated).

> ### 🔴 Why this must NOT be production's instance
>
> `src/graph/repositories/user.repository.ts:20-26` — `createUser()` opens with:
>
> ```cypher
> OPTIONAL MATCH (orphan:User {email: $email})
> WHERE orphan.id <> $id
> DETACH DELETE orphan
> ```
>
> It matches by **email** and deletes any `:User` whose id differs. `users.service.ts:46` calls
> it on **every sign-in**. A tester signing into staging with their real Google account gets a
> new staging uuid, the clause matches their **production** node, and their real bookmarks,
> streak relationships and session history are deleted.

> ### Trap: Neo4j cannot be disabled by env
>
> `src/config/neo4j.config.ts:4` is `process.env.NEO4J_URI || 'bolt://localhost:7687'`. An empty
> or unset value falls through to the **localhost default**, so `uri` is always truthy and the
> "configuration is missing" early-return at `neo4j.service.ts:39` is unreachable. The app will
> always attempt a connection. And if anything *is* listening on `:7687` in that environment, you
> silently get a real connection to the wrong database.
>
> Running without Neo4j is not a viable QA option anyway — `GET /pyq/:id/details` (the
> tap-a-question screen) returns **500**, `POST /pyq/:id/submit` returns 500 **and loses the
> attempt from Postgres too** (the `PyqAttempt` mirror is written after the graph call), all three
> bookmark toggles 500, and mains evaluation is discarded. List endpoints do degrade cleanly —
> which is worse, because they silently show everything as unbookmarked.

## A3. Razorpay test-mode plans

1. Razorpay Dashboard → switch to **Test Mode**.
2. **Settings → API Keys** → generate test key id + secret (`rzp_test_…`).
3. **Subscriptions → Plans** → create four plans:

   | Exam | Period | Amount |
   |---|---|---|
   | upsc-cse | monthly | ₹599 |
   | upsc-cse | annual | ₹5,900 |
   | appsc-group-1 | monthly | ₹599 |
   | appsc-group-1 | annual | ₹5,900 |

4. Fill `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_PLAN_MONTHLY`, `RAZORPAY_PLAN_ANNUAL`.
5. **Settings → Webhooks** → add `https://staging-api.prepmonkey.com/api/v1/payments/webhooks/razorpay`
   (test mode), copy its secret into `RAZORPAY_WEBHOOK_SECRET`.

   ✅ **Expected:** all four plan ids differ from the live ids recorded in
   `memory/project_per_exam_entitlements.md`. Live and test plan ids are **not** interchangeable —
   a live id used against test keys fails with *"invalid or could not be found."*

## A4. DNS + TLS (Cloudflare + ACM)

Android has cleartext disabled, so a bare `http://` ALB hostname will not work for the app.

1. **ACM** (region `ap-south-1`) → Request public certificate → `staging-api.prepmonkey.com` →
   **DNS validation**. ACM shows a `_xxxx.staging-api` CNAME.
2. **Cloudflare → DNS** → add that CNAME exactly, **DNS only (grey cloud)**.
   A proxied record breaks ACM validation.

   ✅ **Expected:** ACM status becomes **Issued**, usually within minutes.
3. Attach the certificate to the staging ALB's **HTTPS:443** listener; redirect 80 → 443.
4. **Cloudflare → DNS** → `CNAME  staging-api  →  <alb-dns-name>`, **DNS only (grey cloud)**.

   ✅ **Expected:** `curl -sS https://staging-api.prepmonkey.com/api/v1/` returns
   `{"message":…,"version":…}` with no TLS warning.

> **Do not use the orange-cloud proxy.** The free plan's ~100s timeout and response buffering
> break the Dify **SSE chat stream**; blocking Dify calls already die at a 60s proxy idle timeout.

> `prepmonkey.com` also serves the deep-link hosts (`go.`, `app.`). Confirm nothing collides with
> AASA / assetlinks.

## A5. Mau service

Same VPC/subnets as the RDS and Valkey, with a security group permitted to reach both. One
replica is enough.

---

# PART B — what is in the repo

| File | Tracked? | Effect on a **production** `mau deploy` |
|---|---|---|
| `.env.staging` | **No** (`.gitignore:39` = `.env*`) | Lands in the image as an inert extra file; never becomes `.env` |
| `Dockerfile` (one ARG) | **Yes** | **None** — default is still `.env.production` |
| `scripts/staging-db.ts` | **Yes** | Inert, never imported by `src/` |
| `scripts/staging-prisma.ts` | **Yes** | Inert, never imported by `src/` |
| `scripts/seed-staging-exam-plans.ts` | **Yes** | Inert, never imported by `src/` |
| `scripts/verify-neo4j-isolation.ts` | **Yes** | Inert, never imported by `src/` |
| `scripts/clone-content-to-staging.ts` | **Yes** | Inert, never imported by `src/` |
| `docs/staging.md` | **Yes** | Inert |

The Dockerfile change:

```dockerfile
ARG ENV_FILE=.env.production
COPY ${ENV_FILE} .env
```

Production is unchanged by construction. Staging overrides it at build time (Part C).

**Why this exists:** `mau` builds the image from the **local working directory**, not git. Without
the ARG, a staging deploy from this tree would bake **production secrets** into the test image.

---

# PART C — bring-up

### C1. Deploy

```bash
# The tree is what ships. Check it first — anything uncommitted goes into the image,
# including work you did not author.
git status

mau deploy --dockerfile Dockerfile --build-arg ENV_FILE=.env.staging
```

✅ **Expected:** `curl https://staging-api.prepmonkey.com/api/v1/health` → `{"status":"ok"}`.

- If the request **hangs** instead of failing → `REDIS_URL` is wrong. Go back to A1.
- Replicas roll gradually. **Probe a new route 15× and wait for 15/15** before judging anything;
  mid-roll results are meaningless.
- If `docker login` fails: either the `aws` CLI is broken by a brew upgrade, or it is the
  OrbStack keychain `-25299` issue.

### C2. Schema — replay and diff

```bash
npx ts-node -T scripts/staging-prisma.ts migrate deploy
npx ts-node -T scripts/staging-prisma.ts drift
```

✅ **Expected: `No difference detected.`**

The drift baseline in this repo is **ZERO** — a full-history replay was proven clean on
2026-08-25. **Any drift is a real bug.** Do not run `prisma migrate dev` against drift: it would
generate a migration that DROPS whatever exists in the database but is missing from
`schema.prisma`. That exact mistake would have dropped 5 GIN indexes.

### C3. Content mirror

```bash
npx ts-node -T scripts/clone-content-to-staging.ts --dry-run   # print the plan
npx ts-node -T scripts/clone-content-to-staging.ts
```

Production is read-only here (`pg_dump` only); `psql` points solely at staging. The script
refuses if the target host looks like production.

Requires `pg_dump` at least as new as the RDS server major — `brew install postgresql@16` if the
installed client is older.

**exam_date:** `exams.exam_date` is the authoritative per-exam field and is what `/config`
serves; `app_config.exam_date` is legacy. Both cloned exam dates are already in the future
(`upsc-cse` 2027-05-23, `appsc-group-1` 2026-11-15), so nothing needs changing. Verified
against prod, which reports the same `daysToExam: 256`.

### C4. `exam_plans` — seed, do not clone

`exam_plans` is deliberately excluded from the mirror. `apple_product_id` and `razorpay_plan_id`
are both `@unique` and hold live store identifiers.

- `razorpay_plan_id` → the **test-mode** ids from A3.
- `apple_product_id` → **copy verbatim.** Apple's sandbox shares the production App Store Connect
  catalog, and `@unique` is per-database.

Insert plans **before** flipping `exams.accessTier` to `paid` — SME refuses `paid` with no active
plan, and a paid exam with no plan is content that is visible but unbuyable.

✅ **Expected:** `GET /paywall` returns plans whose `razorpayPlanId` matches none of the four live
ids.

### C5. Clients

- **`prepmonkey-web`** — point the API base-URL env var at staging on a preview/branch deploy.
  Vercel Hobby blocks deploys whose commit author is not the account owner.
- **`upsc_app`** — a `staging` buildType/flavor overriding **only** the API base URL. Cognito
  config is unchanged; see below. Must be unreachable from a release build.

**Cognito is reused from production on purpose.** It is only the identity provider: the app posts
an ID token to `/auth/cognito/exchange`, and the backend verifies it and mints its own JWT,
writing a `UserAuth` row. All state lives in Postgres, so a tester signing in with the same Google
account is a completely separate user in staging. The isolation comes from the database. This is
what lets the app point at staging by changing only the base URL — no second Google OAuth client,
no second Apple Service ID. `JWT_SECRET` and `REFRESH_TOKEN_SECRET` are freshly generated, so a
production token is invalid here and vice versa.

---

# Verification

1. **Boot** — `/api/v1/health/full`: Postgres, Valkey and Neo4j all green.
2. **Redis isolation** 🔴 — the endpoint is not `upsc-valkey`; keys appear only under
   `{bullque-staging}`.
3. **Neo4j isolation** — ✅ **CONFIRMED 2026-09-09.** Re-run any time:

   ```bash
   npx ts-node -T scripts/verify-neo4j-isolation.ts
   ```

   Read-only on both instances; refuses if the two URIs match. Result: a staging sign-in as
   `yashwanth@stanzasoft.com` left production's `:User` node alive with **15 relationships**
   and streak intact (prod graph still 17,953 nodes). The `DETACH DELETE orphan` clause ran
   against staging's own graph.

   ✅ **The fresh-graph bug this check exposed is FIXED (2026-09-10).**
   `user.repository.ts` used to open `createUser()` with a hard
   `MATCH (t:Tier {id: 'free'})`, but the Neo4j exit had already deleted the endpoint that
   seeded those Tier nodes (`health.controller.ts`) and the `TierNode` types
   (`graph/types/nodes.types.ts`) — leaving the cypher as the only survivor. With no `:Tier`
   nodes the MATCH returned zero rows, the MERGE never ran, and **no `:User` was created**,
   silently, because `users.service.ts` catches and only warns. Production was unaffected
   only because its Tier nodes predate the removal; a restore or new instance would have hit
   it. The `MATCH` and the `HAS_TIER` MERGE are now gone — a fresh graph needs no seeding,
   verified against a Tier-less staging graph.

4. **Schema** — C2 printed "No difference detected."
5. **No outbound leakage** — after 30 minutes of uptime (covering the `*/3` and `*/15` ticks):
   zero new production Razorpay/Apple orders, zero new Wylto CRM contacts, zero unexpected FCM
   sends. Production CloudWatch (container `upsc`) shows nothing new.
6. **Mux safety** — delete a reel in staging; confirm it still plays in production.
7. **QA flow** — sign in → phone OTP → onboarding → open a PYQ → submit → bookmark → mains answer
   + evaluation → chat (SSE) → paywall with test-mode Razorpay.
8. **Then** set `NOTIFICATION_ENGINE_ENABLED=true` and watch a rule tick against zero device
   tokens.

---

# Traps

**Three scheduled jobs default to ON**, and all three flags are absent from `.env.production`, so
they are live in production and would be live in any clone. `.env.staging` sets all three to
`false`:

| Job | Schedule | Flag | Effect if left on |
|---|---|---|---|
| `reconcile-sweep` | `*/3 * * * *` | `PAYMENT_RECONCILE_ENABLED` | Calls live Razorpay + Apple; grants entitlements |
| `expiry-sweep` | `0 * * * *` | same | Downgrades users |
| notification tick | `*/15 * * * *` | `NOTIFICATION_ENGINE_ENABLED` | Sends real FCM pushes |
| usage rollup / purge | `0 19/20 * * *` | `API_USAGE_ROLLUP_ENABLED` | Purge deletes rows |

`WYLTO_CONTACT_SYNC_ENABLED` is opt-in and left unset — enabling it pushes to the **production**
Wylto CRM. `WYLTO_WEBHOOK_SECRET` is left unset so the webhook guard fails closed.

**Mux tokens are dummies on purpose.** `MuxService` is boot-fatal so the vars must exist, but
playback needs no Mux API — `playbackId` is a stored column and clients stream from Mux's CDN.
Every Mux call site is ingest or **delete**: with production tokens, deleting a reel in staging
would delete the real production video asset.

**Never put a database URL in a shell command.** It mangles into a misleading *"invalid port
number in database URL"*, and echoing it to debug is blocked. Both scripts read the URL inside
Node and pass it via an argv array with `shell:false`.

**Boot-fatal variables** — the app `process.exit(1)`s without these. Real and reachable:
`DATABASE_URL`, `REDIS_URL`. Merely present and well-formed: `JWT_SECRET`,
`REFRESH_TOKEN_SECRET`, `COGNITO_USER_POOL_ID` (must match `/^[a-z]{2}-[a-z]+-\d_[a-zA-Z0-9]+$/`),
`RAZORPAY_KEY_ID`/`_SECRET`, `MUX_TOKEN_ID`/`_SECRET`. Also the three Apple root `.cer` files at
`src/modules/payments/certs/apple/`, carried into `dist` by `nest-cli.json:14-18`.

---

# Unrelated findings (not addressed here)

- **`README.md` is tracked and contains live API key/secret pairs** — both the production pair and
  the newly added test pair. `API_KEY_SECRET` gates the entire `/sme/*` management API. The values
  are in git history and remain compromised until rotated. Left untouched by request.
- **The Valkey `noeviction` instructions were deleted from `README.md`** in an uncommitted change.
  They are a real production requirement and are reproduced in A1 above.
- `README.md` is modified and `scripts/usage-report/` is untracked in the working tree. Neither is
  schema-affecting, but both would ship on any `mau deploy` from this tree.

---

# 2026-09-17 per-exam config: staging verification checklist

Everything the per-exam-config work (flags, limits, trial, paywall, banners, quota UX) and the
two fixes before it must prove **on staging, in this order, before any of it is deployed to
production**. Each step states the exact request and the result that counts as a pass.

Shorthand used below:

```bash
STG=https://staging-api.prepmonkey.com
KEY="<API_KEY_SECRET for staging>"          # x-api-key on every /sme/* call
npx ts-node -T scripts/staging-token.ts --out /tmp/staging.jwt
TOKEN="$(tr -d '\n' < /tmp/staging.jwt)"     # never echo this
```

`staging-db.ts` is the query runner — **do not write another one-off script**:

```bash
npx ts-node -T scripts/staging-db.ts "SELECT …"
npx ts-node -T scripts/staging-db.ts --write "UPDATE …"
```

## Step 0 — REDEPLOY FIRST, then prove what is running

🔴 `mau` builds the image from the **local working tree, not git**. A commit proves nothing
about what is live, and an uncommitted file ships. Nothing below is trustworthy until the
running image is confirmed **through the API**.

```bash
git status                 # anything uncommitted goes into the image

# ⚠️ FIRST: capture the back-compat baseline off the OLD image. Once you deploy,
# there is nothing left to compare against. This is step 5's "before".
npx ts-node -T scripts/verify-exam-backcompat.ts --base-url "$STG" --token "$TOKEN" --out /tmp/pre.json

mau deploy --dockerfile Dockerfile --build-arg ENV_FILE=.env.staging
```

Then, and only then:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -H "x-api-key: $KEY" \
  "$STG/api/v1/sme/exams/feature-registry"
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $TOKEN" \
  "$STG/api/v1/banners?platform=android"
```

✅ **Expected: `200` from both, 15 times in a row.** Both routes exist only in this build, so a
`404` means the old image is still serving. Replicas roll gradually — **probe 15×, require
15/15** before judging anything.

## Step 1 — Apply the three migrations by hand

```bash
npx ts-node -T scripts/staging-prisma.ts migrate deploy
npx ts-node -T scripts/staging-prisma.ts drift
```

Applying: `20260916000000_mains_evaluations_and_attempt_dedupe`,
`20260917000000_per_exam_config`, `20260917000001_exam_banners`.

✅ **Expected: `No difference detected.`** The drift baseline in this repo is **zero**; any
drift is a real bug. Then confirm the seed the migration performs:

```bash
npx ts-node -T scripts/staging-db.ts \
  "SELECT id, trial_days, trial_policy FROM exams ORDER BY id"
```

✅ **Expected:** `upsc-cse` → `trial_days = 14`, policy is the single epoch entry.
`appsc-group-1` → `trial_days = 0`, policy has **two** entries (`14` at epoch, `0` at the
migration instant).

Then the flag backfill, which is what keeps APPSC from silently gaining Mains/Reels/Library
on deploy (a NULL override blob resolves to all-true):

```bash
npx ts-node -T scripts/staging-db.ts \
  "SELECT id, enabled_modules, feature_flags FROM exams ORDER BY id"
```

✅ **Expected:** `upsc-cse` → `feature_flags` **NULL** (it already has all five modules, so
there is nothing to override — byte-identical to before). `appsc-group-1` →
`{"mains": false, "reels": false, "library": false, "library.simulation": false}`, which
mirrors straight back to `enabled_modules = {prelims}`. An exam with an EMPTY
`enabled_modules` also stays NULL — empty already means "show everything" on every client.

## Step 2 — Razorpay renewal webhook, in the UNWRAPPED shape, actually grants

The bug: `handleSubscriptionEvent` read `payload.payload.subscription.entity` while the
controller already unwraps once, so every real renewal logged one warn, returned 200, was
stamped `processed`, and granted nothing while the card was debited.

```bash
SECRET="$(grep -m1 '^RAZORPAY_WEBHOOK_SECRET=' .env.staging | cut -d= -f2-)"
BODY='{"event":"subscription.charged","payload":{"subscription":{"entity":{"id":"<REAL staging sub_ id>","status":"active","current_end":1790000000}},"payment":{"entity":{"id":"pay_stgtest001","amount":590000,"currency":"INR","status":"captured"}}}}'
SIG="$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$SECRET" -r | cut -d' ' -f1)"
curl -sS -X POST "$STG/api/v1/webhooks/razorpay" \
  -H 'Content-Type: application/json' -H "x-razorpay-signature: $SIG" -d "$BODY"
```

✅ **Expected:** HTTP 200 **and** the grant actually happened:

```bash
npx ts-node -T scripts/staging-db.ts \
  "SELECT external_id, type, result, processed_at FROM payment_events WHERE provider='razorpay' ORDER BY received_at DESC LIMIT 3"
npx ts-node -T scripts/staging-db.ts \
  "SELECT user_id, exam_id, expires_at, revoked_at FROM user_exam_entitlements ORDER BY updated_at DESC LIMIT 3"
npx ts-node -T scripts/staging-db.ts \
  "SELECT id, order_id, amount, status FROM payments ORDER BY created_at DESC LIMIT 3"
```

An event stamped with a `processed_at` and `result` other than `granted`, with **no** new
`payments` row and **no** entitlement extension, is the exact failure this step exists to
catch — a 200 alone is not a pass.

## Step 3 — `POST /pyq/:id/submit` persists to Postgres first, and survives a graph failure

```bash
curl -sS -X POST "$STG/api/v1/pyq/<QUESTION_ID>/submit" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"selectedOption":"A"}'
npx ts-node -T scripts/staging-db.ts \
  "SELECT user_id, question_id, attempt_number, is_correct, created_at FROM pyq_attempts ORDER BY created_at DESC LIMIT 3"
```

✅ **Expected:** 2xx, and the row is in Postgres. (`SubmitAttemptDto` accepts **only**
`selectedOption` — the global pipe runs `forbidNonWhitelisted`, so any extra key is a 400.)

Then repeat with Neo4j unreachable (point `NEO4J_URI` at `bolt://127.0.0.1:1` and redeploy, or
pause the Aura instance):

✅ **Expected: still 2xx, still a Postgres row.** The graph write is additive and best-effort;
a graph outage must never lose the attempt or 500 the request. Restore `NEO4J_URI` afterwards.

## Step 4 — A mains evaluation lands in `mains_evaluations`

```bash
curl -sS -X POST "$STG/api/v1/mains/<MAINS_ID>/evaluation" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"score":7.5,"marks":8,"evaluationMarkdown":"## Feedback\n…","timeTaken":420}'
npx ts-node -T scripts/staging-db.ts \
  "SELECT id, user_id, question_id, attempt_number, score, marks, created_at FROM mains_evaluations ORDER BY created_at DESC LIMIT 3"
```

✅ **Expected:** the row exists in **Postgres** (the table `20260916000000` added), not only in
the graph.

## Step 5 — `verify-exam-backcompat.ts --compare` is IDENTICAL

Captured **before** the deploy in step 0 and again after it:

```bash
# before the deploy
npx ts-node -T scripts/verify-exam-backcompat.ts --base-url "$STG" --token "$TOKEN" --out /tmp/pre.json
# after (fully rolled)
npx ts-node -T scripts/verify-exam-backcompat.ts --base-url "$STG" --token "$TOKEN" \
  --out /tmp/post.json --compare /tmp/pre.json
# and the no-header ≡ X-Exam: upsc-cse half
npx ts-node -T scripts/verify-exam-backcompat.ts --base-url "$STG" --token "$TOKEN" \
  --exam upsc-cse --out /tmp/post-upsc.json --compare /tmp/post.json
```

✅ **Expected: `✓ IDENTICAL` on both compares, exit 0**, and **no** "absent from the baseline
and therefore NOT compared" warning on the second one. The probe list now covers `/config`,
`/exams`, `/paywall?platform=android`, `/paywall/banner?platform=android`,
`/banners?platform=android`, `/quota/me` and `/pyq/:id/details` alongside the original ~24.

⚠️ A capture full of 401s/404s will happily "match". The script warns; do not ignore it.

## Step 6 — Onboarding completes on device, reports success, and fires trial sync + ATT

On a real device (staging build): sign in → finish onboarding.

✅ **Expected:** the completion call returns success and the app advances to the dashboard;
logcat/Console shows the trial-sync call and, on iOS, the ATT prompt — and **not** on cold
launch. Confirm server-side:

```bash
npx ts-node -T scripts/staging-db.ts \
  'SELECT "userId", active_exam_id, "onboardingCompleted" FROM user_profiles ORDER BY "updatedAt" DESC LIMIT 3'
```

⚠️ On `user_profiles`, only `active_exam_id` is snake_cased — `userId`, `updatedAt` and
`onboardingCompleted` have no `@map`, so those columns really are camelCase and must be
double-quoted in SQL.

## Step 7 — A long question renders all four options and the reveal link (uiautomator)

```bash
adb shell uiautomator dump /sdcard/ui.xml && adb pull /sdcard/ui.xml /tmp/ui.xml
grep -o 'text="[A-D][).] [^"]\{0,40\}' /tmp/ui.xml
```

✅ **Expected:** four option nodes **and** the reveal control present in the dump, on a
question whose stem overflows the viewport. A screenshot is not enough — the point is that
nothing is clipped out of the tree.

## Step 8 — 403 `FEATURE_DISABLED` on an APPSC-disabled route; the same route with no header is 200

```bash
curl -sS -X PATCH "$STG/api/v1/sme/exams/appsc-group-1" \
  -H "x-api-key: $KEY" -H 'Content-Type: application/json' \
  -d '{"featureFlags":{"reels.readMore":false}}'
sleep 65                      # the catalogue snapshot is 60s PER REPLICA

curl -s -o /tmp/a.json -w '%{http_code}\n' -H "Authorization: Bearer $TOKEN" \
  -H 'X-Exam: appsc-group-1' "$STG/api/v1/reels/<REEL_ID>/blog"
curl -s -o /tmp/b.json -w '%{http_code}\n' -H "Authorization: Bearer $TOKEN" \
  "$STG/api/v1/reels/<REEL_ID>/blog"
cat /tmp/a.json
```

✅ **Expected:** `403` then `200`, and the 403 body carries
`{"code":"FEATURE_DISABLED","feature":"reels.readMore","exam":"appsc-group-1"}` alongside
`"error":"Forbidden"` — **not** a flattened generic error, and **never** a 402.

Also check the derived mirror and the resolved map:

```bash
curl -sS -H "x-api-key: $KEY" "$STG/api/v1/sme/exams/appsc-group-1" | jq '.effectiveFeatureFlags["reels.readMore"], .enabledModules'
```

Then **put it back**: `{"featureFlags":{"reels.readMore":true}}` (`true` removes the override).

## Step 9 — A fresh APPSC user is `free`, not `trial`

Create a brand-new staging account, then:

```bash
curl -sS -H "Authorization: Bearer $NEW_TOKEN" -H 'X-Exam: appsc-group-1' \
  "$STG/api/v1/payments/entitlement" | jq '{isTrial, trialEndsAt, trialDaysLeft, isPremium}'
curl -sS -H "Authorization: Bearer $NEW_TOKEN" -H 'X-Exam: appsc-group-1' \
  "$STG/api/v1/quota/me" | jq '.premium, .features.pyq_reveal'
```

✅ **Expected:** `isTrial: false` for APPSC, and `/quota/me` reports `premium: false` with a
real metered `pyq_reveal` entry (limit + remaining + `resetsAt`). The **same user with no
header** (i.e. UPSC) must still show the 14-day trial — the clock is person-level, the length
is per exam.

## Step 10 — A per-exam cap is honoured, and UPSC is unaffected

```bash
curl -sS -X PATCH "$STG/api/v1/sme/exams/appsc-group-1" \
  -H "x-api-key: $KEY" -H 'Content-Type: application/json' \
  -d '{"featureCaps":{"pyq_reveal":{"type":"daily","limit":5}}}'
sleep 65
```

As a **free** APPSC user, reveal four different questions:

```bash
for q in Q1 Q2 Q3 Q4; do
  curl -s -o /dev/null -w "$q %{http_code}\n" -H "Authorization: Bearer $NEW_TOKEN" \
    -H 'X-Exam: appsc-group-1' "$STG/api/v1/pyq/$q/reveal"
done
```

✅ **Expected:** all four `200` (the 4th is the one that used to 402), with
`X-Quota-Limit: 5`. The same user's UPSC reveals must still cap at **3** — the 4th with no
exam header returns `402 QUOTA_EXHAUSTED` carrying `limit: 3` and a `resetsAt`.

Then check the paywall reflects it (this exam is now "configured"):

```bash
curl -sS -H "Authorization: Bearer $NEW_TOKEN" -H 'X-Exam: appsc-group-1' \
  "$STG/api/v1/paywall?platform=android" | jq '.comparison'
```

✅ **Expected:** a GENERATED table whose "Answer explanations" row reads `5/day`, not `3/day`.
Clear it afterwards with `{"featureCaps":{"pyq_reveal":null}}`.

## Step 11 — Reveal idempotency: a second reveal of the same question does not decrement

```bash
curl -s -D- -o /dev/null -H "Authorization: Bearer $NEW_TOKEN" "$STG/api/v1/pyq/Q9/reveal" | grep -i x-quota-remaining
curl -s -D- -o /dev/null -H "Authorization: Bearer $NEW_TOKEN" "$STG/api/v1/pyq/Q9/reveal" | grep -i x-quota-remaining
curl -sS -H "Authorization: Bearer $NEW_TOKEN" "$STG/api/v1/pyq/Q9/details" | jq '.unlockedToday, .canViewExplanation'
```

✅ **Expected:** the **same** `X-Quota-Remaining` on both calls (a credit is spent once per
question per IST day), and `/details` reports `unlockedToday: true` with the explanation
inline. This is the fix for the production "I can't access my content" reports.

## Step 12 — `GET /banners` resolves by tier × platform × account age

Create a banner targeted at exactly the test user's standing, activate it, and read it back:

```bash
curl -sS -X POST "$STG/api/v1/sme/banners" -H "x-api-key: $KEY" -H 'Content-Type: application/json' -d '{
  "examId":"upsc-cse","name":"staging check",
  "imageUrl":"https://example.invalid/x.png",
  "destinationUrl":"https://go.prepmonkey.com/open/paywall",
  "audienceTiers":["trial"],"platforms":["android"],
  "minAccountAgeDays":0,"maxAccountAgeDays":3650,"priority":50,
  "startsAt":"2026-09-01T00:00:00+05:30","endsAt":"2027-09-01T00:00:00+05:30"}'
curl -sS -X POST "$STG/api/v1/sme/banners/<ID>/activate" -H "x-api-key: $KEY"

curl -sS -H "Authorization: Bearer $TOKEN" "$STG/api/v1/banners?platform=android" | jq '.banners'
curl -sS -H "Authorization: Bearer $TOKEN" "$STG/api/v1/banners?platform=ios"     | jq '.banners'
```

✅ **Expected:** present on `android`, absent on `ios`, and absent again once
`audienceTiers` is PATCHed to `["paid"]` (for a non-paying user). A live campaign banner, if
any, appears alongside it as `"kind":"campaign"` **below** priority 50. Missing `platform`
must be a 400, not a guess. Finish with `POST /sme/banners/<ID>/end`.

Then prove the flag gate: `PATCH /sme/exams/upsc-cse {"featureFlags":{"dashboard.banners":false}}`
→ `GET /banners` returns **403 `FEATURE_DISABLED`**, not `[]`. Put it back with `true`.

## Step 13 — Onboarding `examId` persists `active_exam_id`

```bash
curl -sS -X POST "$STG/api/v1/user/profile/onboarding/complete" \
  -H "Authorization: Bearer $NEW_TOKEN" -H 'Content-Type: application/json' \
  -d '{"…the usual onboarding payload…","examId":"appsc-group-1"}'
npx ts-node -T scripts/staging-db.ts \
  'SELECT "userId", active_exam_id FROM user_profiles ORDER BY "updatedAt" DESC LIMIT 3'
curl -sS -H "Authorization: Bearer $NEW_TOKEN" "$STG/api/v1/user/profile/me" | jq '.activeExamId'
```

✅ **Expected:** `active_exam_id = 'appsc-group-1'` — this is the path that closes the
`updateMany` no-op hole in `PUT /exams/active` for a pre-onboarding user. An **unknown** slug
must store `upsc-cse` and still return success; nobody is locked out of finishing onboarding
by a stale id. Because APPSC grants 0 trial days, **no `trial_started` notification** should be
queued for this user.

## Step 14 — Paywall: strike price + generated table for a configured exam; byte-identical for UPSC

```bash
curl -sS -X POST "$STG/api/v1/sme/exams/appsc-group-1/plans" -H "x-api-key: $KEY" \
  -H 'Content-Type: application/json' \
  -d '{"planId":"annual","planType":"ANNUAL","priceInPaise":590000,"strikePriceInPaise":799900}'
curl -sS -H "Authorization: Bearer $TOKEN" -H 'X-Exam: appsc-group-1' \
  "$STG/api/v1/paywall?platform=android" | jq '.plans[] | {id, price, strikePrice, strikePeriod, appleProductId}'
```

✅ **Expected:** `strikePrice: "₹7,999"` with `strikePeriod: "/year"` beside `price: "₹5,900"`,
and — the one that matters — **`appleProductId` and `razorpayPlanId` still populated** after
that price-only upsert. A `strikePriceInPaise` ≤ `priceInPaise` must be a 400.

And for UPSC, which nobody has configured:

```bash
curl -sS -H "Authorization: Bearer $TOKEN" "$STG/api/v1/paywall?platform=android" | jq '.comparison | length'
```

✅ **Expected: 12 rows, the hand-written ones, verbatim** — covered byte-for-byte by step 5's
`paywall-android` probe. If UPSC's table is generated instead, something wrote
`paywall_content` / `feature_flags` / `feature_caps` on `upsc-cse` and must be cleared before
production.

## Before promoting any of this to production

- Re-run step 5 against **production**, pre- and post-deploy, and require `✓ IDENTICAL`.
- Confirm nothing was left switched off on `upsc-cse`: `featureFlags`, `featureCaps` and
  `paywallContent` must all still be `null`.
- `git status` must be clean of anything you do not intend to ship — `mau` ships the tree.
