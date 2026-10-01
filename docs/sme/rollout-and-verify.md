# SME Management — Rollout & Verify Checklist

> Changed 2026-09-21 — exam dimension on users, orders, offers, banners; two BREAKING calls (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).
> Changed 2026-09-21 — per-exam broadcast, validated segment exam (+ fix), survey targetExams, feedback ?exam= (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

Covers deploying the SME management work **and** the verify-only critical items
(#1 Wylto runtime env, #5 notification-history migration) folded into this session.

## 1. Rollout ordering (load-bearing)

`mau` builds the image from the **local working dir** (not git), and the new
Prisma client SELECTs/writes new columns. Apply the DB migration BEFORE the
deploy, or auth/payment paths will error. See `feedback_mau_deploy_behavior`,
`feedback_manual_migrations_drift`.

1. Apply migration `prisma/migrations/20260616162003_sme_management`
   (adds `orders.premium_granted_at`, `PaymentSource.MANUAL`, `sme_audit_log`) —
   `npx prisma migrate deploy`, or run its `migration.sql` by hand on prod.
   - `ALTER TYPE ... ADD VALUE 'MANUAL'` needs PG 12+ (we add but don't use it in
     the same migration, so it commits cleanly).
2. `npx prisma generate`.
3. `mau deploy` (ensure `API_KEY_SECRET` is set in the mau runtime env — it gates
   every `/sme/*` route).
4. Smoke: `GET /api/v1/sme/users?limit=1` with `x-api-key: $API_KEY_SECRET` → 200;
   without the header → 401.

## 2. Verify-only item #5 — notification-history migration on prod

Check whether the in-app feed tables exist on prod (gates topics + bell feed):

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_name IN ('notification_history', 'topic_notifications');
-- Expect BOTH rows. If missing, apply the notifications migration before relying
-- on the in-app feed / SME broadcast persistence.
```

## 3. Verify-only item #1 — Wylto runtime env + reconcile schedule

Confirm in the **mau runtime env** (not just `.env.production`):

```bash
# In the running task / mau env inspection:
#   WYLTO_CONTACT_SYNC_ENABLED=true
#   WYLTO_MARKETING_API_KEY=<present>
```

In the boot logs after deploy, confirm the reconcile registered:

```
Wylto daily reconcile scheduled (0 21 * * * UTC)
```

Then let the first nightly reconcile fire (21:00 UTC = 02:30 IST) and confirm it
scans without errors. See `project_wylto_contact_sync`.

## 4. SME endpoint smoke (post-deploy, with x-api-key)

- `POST /sme/users/:id/premium {plan:"MONTHLY"}` → user flips `SUBSCRIBED`, an
  Order(source=MANUAL)+Payment(CAPTURED) appears in `GET /sme/transactions`.
- ~~`DELETE /sme/users/:id/premium` → back to `UNSUBSCRIBED`, Neo4j tier free.~~
  **Superseded by §7.1/§7.2** — the bare call is now a 400, and revoking one exam correctly
  leaves a user who holds another on `SUBSCRIBED`.
- `POST /sme/users/:id/deactivate` → `SUSPENDED`; `DELETE /sme/users/:id` → cascade purge.
- Webhook reconcile (#3): a `payment.captured` for a PAID-but-not-granted order
  grants premium once; redelivery is a no-op (`order.premium_granted_at` guard).
- Each destructive/privileged call writes an `sme_audit_log` row.

---

## 6. Notification engine (added 2026-08-26)

**Ordering is load-bearing — two of these fail SILENTLY if skipped.**

1. **Apply both migrations by hand, before the deploy.**
   `20260826000000_streak_and_pyq_postgres_mirrors`, then
   `20260826010000_notification_engine`. Both additive and idempotent.

2. **Verify against `information_schema`, not `_prisma_migrations`** (migrations are applied by
   hand here, so the migrations table is not trustworthy):

```sql
SELECT
  (SELECT count(*) FROM information_schema.tables  WHERE table_name='daily_sessions')                                   AS daily_sessions,
  (SELECT count(*) FROM information_schema.tables  WHERE table_name='pyq_attempts')                                     AS pyq_attempts,
  (SELECT count(*) FROM information_schema.tables  WHERE table_name='notification_rule')                                AS notification_rule,
  (SELECT count(*) FROM information_schema.columns WHERE table_name='user_auth' AND column_name='current_streak')       AS current_streak,
  (SELECT count(*) FROM information_schema.columns WHERE table_name='app_config' AND column_name='notification_config') AS notification_config;
-- Expect 1 on every column.
```

3. **Seed the rules** — `scripts/seed-notification-rules.ts --execute` (dry run first).
   Creates 13 rules; only `payment_success` and `trial_ending` enabled, because those two
   already fire in production. **Skipping this stops payment confirmations**, with nothing in
   the logs except the boot alarm in step 6.

4. **Backfill streak + PYQ attempts BEFORE the deploy** —
   `scripts/backfill-streak-and-pyq-from-neo4j.ts --prod` (dry run), then `--prod --execute`.
   At deploy the streak endpoints flip their reads to Postgres; if the tables are empty every
   user with session history sees an empty progress calendar.
   ⚠️ Use `--prod`, do NOT pass `DATABASE_URL=` on the command line — extracting it in the
   shell mangles the password and Prisma fails with a misleading "invalid port number". That
   is why the first prod run of this script wrote zero rows.

5. **Deploy**, then re-run the backfill once to pick up the gap.

6. **Read the boot log.** If you see `[NOTIFICATION GAP]`, step 3 did not happen and users are
   paying without being told.

7. **Smoke:**

```bash
# 401 (route exists, wants a key) — NOT 404
curl -s -o /dev/null -w '%{http_code}\n' https://app.stanzasoft.ai/api/v1/sme/notification-rules/triggers
# 13 triggers, exactly 2 rules live
curl -s -H "x-api-key: $KEY" https://app.stanzasoft.ai/api/v1/sme/notification-rules \
  | jq -r '.data[] | select(.isLiveNow) | .triggerKey'
```

8. **Confirm the scheduler is ticking** — after the first rule's send time passes, a row should
   exist in `notification_rule_run`. Disabled rules appear there in `SHADOW` mode with
   `sent = 0`; that is success, not failure.

---

## 7. Exam dimension + the two breaking calls (added 2026-09-21)

Backend `4f6622e`. Everything here is **read-path additive** except the two calls in §7.1,
which return 400 where they used to succeed. Run §7.1 **first** — if the portal is still making
the old calls, you want to know before anyone tries to process a refund.

Set up once:

```bash
KEY=$API_KEY_SECRET
BASE=https://app.stanzasoft.ai/api/v1
UID=<a test user id>
```

### 7.1 Breaking-change smoke — both 400s, by their exact message

```bash
# 1. Bare revoke must now be refused.
curl -s -X DELETE -H "x-api-key: $KEY" "$BASE/sme/users/$UID/premium" | jq -r '.message'
# EXPECT exactly:
#   exam is required: pass ?exam=<examId> to revoke one exam, or ?exam=all to revoke every exam.

# 2. Offer create without examId must be refused.
curl -s -X POST -H "x-api-key: $KEY" -H 'content-type: application/json' \
  -d '{"code":"SMOKE400","name":"smoke","startsAt":"2030-01-01T00:00:00+05:30",
       "endsAt":"2030-01-02T00:00:00+05:30","plans":[{"id":"yearly","title":"Y","price":"₹1","period":"/year"}],
       "content":{}}' \
  "$BASE/sme/offers" | jq -r '.message'
# EXPECT exactly:
#   examId is required (e.g. upsc-cse). Campaigns are per exam.
```

🔴 **If either returns 2xx, the deploy did not take** — you are on the old image. Do not
proceed; a bare revoke on the old build destroys every exam's entitlement for that user.

Also confirm an unknown slug 400s rather than returning an empty page, on each new parameter:

```bash
for q in "sme/users?exam=nope" "sme/orders?exam=nope" "sme/offers?exam=nope" \
         "sme/banners?exam=nope" "sme/users/$UID/incidents?exam=nope"; do
  printf '%s -> %s\n' "$q" "$(curl -s -o /dev/null -w '%{http_code}' -H "x-api-key: $KEY" "$BASE/$q")"
done
# EXPECT 400 on every line. A 200 with an empty list means the strict check is missing.
```

### 7.2 Per-exam grant/revoke — the whole point of the change

Use a test user who holds **two** exams, or build one. The assertion is that touching one exam
leaves the other **untouched**.

1. Grant both:
   ```bash
   curl -s -X POST -H "x-api-key: $KEY" -H 'content-type: application/json' \
     -d '{"plan":"MONTHLY","examId":"upsc-cse"}'      "$BASE/sme/users/$UID/premium" | jq
   curl -s -X POST -H "x-api-key: $KEY" -H 'content-type: application/json' \
     -d '{"plan":"MONTHLY","examId":"appsc-group-1"}' "$BASE/sme/users/$UID/premium" | jq
   ```
2. **Record the before-picture** — two `LIVE` rows:
   ```bash
   curl -s -H "x-api-key: $KEY" "$BASE/sme/users/$UID" \
     | jq -c '{status, isPremium, ent: [.entitlements[] | {examId, status}]}'
   # EXPECT: status SUBSCRIBED, isPremium true, both exams LIVE
   ```
3. Revoke **one** exam:
   ```bash
   curl -s -X DELETE -H "x-api-key: $KEY" \
     "$BASE/sme/users/$UID/premium?exam=appsc-group-1" | jq
   # EXPECT: { "success": true, "examId": "appsc-group-1", "expiredAt": "…" }
   ```
4. **The assertion.** Re-read the user:
   ```bash
   curl -s -H "x-api-key: $KEY" "$BASE/sme/users/$UID" \
     | jq -c '{status, isPremium, ent: [.entitlements[] | {examId, status}]}'
   ```
   - ✅ `appsc-group-1` → `REVOKED`
   - ✅ `upsc-cse` → still `LIVE`, **untouched**
   - ✅ `status` is **still `SUBSCRIBED`** and `isPremium` still `true` — a user who holds
     another exam stays a paying customer, and the mirror is recomputed from what survives
     rather than asserted to `UNSUBSCRIBED`
   - 🔴 If UPSC also went `REVOKED`, or `status` flipped to `UNSUBSCRIBED`, **stop and roll
     back**: that is the pre-`4f6622e` behaviour and it is destroying real subscriptions.
5. `?exam=all` still does the old thing, on purpose. Verify separately, on a throwaway user:
   ```bash
   curl -s -X DELETE -H "x-api-key: $KEY" "$BASE/sme/users/$THROWAWAY/premium?exam=all" | jq
   # EXPECT: { "success": true, "examId": null, "expiredAt": "…" }  ← null means ALL, not none
   ```
6. Clean up the test grants with `?exam=all`.

### 7.3 The two-axis user filter

```bash
# Axis 1 — who they ARE (active_exam_id). Note upsc-cse ALSO matches NULL.
curl -s -H "x-api-key: $KEY" "$BASE/sme/users?exam=appsc-group-1&limit=1" | jq '.total'

# Axis 2 — what they HOLD (a live user_exam_entitlements row in THAT exam).
curl -s -H "x-api-key: $KEY" "$BASE/sme/users?exam=appsc-group-1&premium=true&limit=5" \
  | jq -c '.total, [.data[] | {id, activeExamId, ent: [.entitlements[] | select(.status=="LIVE") | .examId]}]'
# EXPECT: every returned row has "appsc-group-1" in its LIVE list.
#         activeExamId may be a DIFFERENT exam or null — that is correct, not a bug.
```

Sanity checks on the same call:
- `?exam=upsc-cse` returns a much larger `total` than the literal count, because it also matches
  users with `active_exam_id` NULL. That is the intended rule, the same one push segments use.
- `?premium=true` **without** `exam` is the unchanged person-level mirror. Its `total` should be
  ≥ the per-exam number.
- Every row carries `activeExamId`, `trialExamId` and `entitlements[]`:
  ```bash
  curl -s -H "x-api-key: $KEY" "$BASE/sme/users?limit=1" \
    | jq -e '.data[0] | has("activeExamId") and has("trialExamId") and has("entitlements")'
  ```

### 7.4 Reconcile buckets

```bash
# The UNRESOLVED_EXAM bucket — paid Apple orders whose product has no exam_plans row.
curl -s -H "x-api-key: $KEY" "$BASE/sme/orders?exam=none&orderStatus=PAID" \
  | jq -c '.total, [.data[] | {id, appleTransactionId, examId, reconcileHint}]'
# Every row should read reconcileHint "UNRESOLVED_EXAM" with examId null.
# These are REAL money with an unknown product: seed exam_plans first, THEN grant with the
# right examId. Never grant one of these blind.
```

Also spot-check that `reconcileHint` is present and that the Apple money fields come through:

```bash
curl -s -H "x-api-key: $KEY" "$BASE/sme/orders?source=APPLE&limit=1" \
  | jq -c '.data[0] | {examId, reconcileHint, amount, currency, storefrontCurrency, chargedAmount, offerCode, listAmount, isTest}'
```

### 7.5 Snapshot / support surface

```bash
curl -s -H "x-api-key: $KEY" "$BASE/sme/users/$UID/snapshot" \
  | jq -c '{activeExamId: .entitlement.activeExamId,
            ents: [.entitlement.entitlements[] | {examId, status}],
            quotaExam: .quota.examId}'
# quota.examId must match the user's activeExamId (upsc-cse when null) — before this release
# the quota panel always reported UPSC.
```

### 7.6 Offers + banners

```bash
curl -s -H "x-api-key: $KEY" "$BASE/sme/offers?exam=upsc-cse" | jq -c '[.data[] | {code, examId, status}]'
curl -s -H "x-api-key: $KEY" "$BASE/sme/banners?exam=upsc-cse" | jq '.total'
```
Two campaigns `LIVE` at once in **different** exams is the intended state — do not treat it as
a data error.

---

## 8. Migrations for this release (2026-09-21) — two of them

**Both are additive and idempotent** (`IF NOT EXISTS` throughout), and migrations here are
applied **by hand**, so re-running either must be a no-op.

| # | Migration | What it does |
|---|---|---|
| 1 | `20260921000000_sme_exam_dimension` | `api_usage.exam_id`, `api_usage_daily.exam_id` (+ **regrained** unique key), `feedback_reports.exam_id`, `surveys.target_exams` |
| 2 | `20260921000001_feedback_exam_idx` | the index behind `GET /sme/feedback/reports?exam=` |

Apply them **in that order**, then deploy.

> 🔴 **Migrate and deploy in ONE window — the rollup job breaks in between.**
> Migration 1 drops the old `api_usage_daily_day_user_id_route_key` and replaces it with a
> four-column key that includes `exam_id`. The **old image's** rollup upserts with
> `ON CONFLICT (day, user_id, route)`, which no longer has a matching unique index, so
> every run **errors** until the new image is serving. `mau` builds from the local working
> dir, so confirm the tree you are deploying is the one that contains both migrations
> before you start. Do not migrate on a Friday and roll the image on Monday.
>
> Migration 1 also backfills `exam_id = 'upsc-cse'` for the **last 3 IST days only** of
> `api_usage_daily` — exactly the window the rollup re-aggregates. Older rows stay NULL on
> purpose (an honest "unknown", not a guessed UPSC). Do not widen that `UPDATE`.

### 8.1 Verify against `information_schema`, not `_prisma_migrations`

```sql
SELECT
  (SELECT count(*) FROM information_schema.columns
     WHERE table_name='api_usage'        AND column_name='exam_id')     AS api_usage_exam_id,
  (SELECT count(*) FROM information_schema.columns
     WHERE table_name='api_usage_daily'  AND column_name='exam_id')     AS api_usage_daily_exam_id,
  (SELECT count(*) FROM information_schema.columns
     WHERE table_name='feedback_reports' AND column_name='exam_id')     AS feedback_reports_exam_id,
  (SELECT count(*) FROM information_schema.columns
     WHERE table_name='surveys'          AND column_name='target_exams') AS surveys_target_exams;
-- Expect 1 on every column.
```

```sql
SELECT indexname
FROM pg_indexes
WHERE schemaname = 'public'
  AND indexname IN (
    'api_usage_exam_id_created_at_idx',
    'api_usage_daily_day_user_id_route_exam_id_key',
    'feedback_reports_exam_id_created_at_idx',
    'api_usage_daily_day_user_id_route_key'   -- the OLD one
  );
-- Expect the FIRST THREE, and NOT the fourth.
```

🔴 **If `api_usage_daily_day_user_id_route_key` is still there, migration 1 did not fully
apply** — the drop and the new key are one step. Leaving both in place makes the new
four-column upsert unreachable for any row whose `(day, user_id, route)` already exists.

Sanity-check the narrow backfill while you are in there:

```sql
SELECT count(*) FILTER (WHERE exam_id IS NULL)        AS unknown_rows,
       count(*) FILTER (WHERE exam_id IS NOT NULL)    AS tagged_rows,
       min(day) FILTER (WHERE exam_id IS NOT NULL)    AS earliest_tagged_day
FROM api_usage_daily;
-- earliest_tagged_day should be ~2 days before today (IST), not the start of history.
```

### 8.2 Post-deploy smoke — broadcast, segment, survey, feedback

```bash
KEY=$API_KEY_SECRET
BASE=https://app.stanzasoft.ai/api/v1
```

**Broadcast — the topic is echoed, and a bad slug 400s.** Use a real but quiet exam, or a
test device; this sends for real.

```bash
curl -s -X POST -H "x-api-key: $KEY" -H 'content-type: application/json' \
  -d '{"title":"smoke","body":"smoke","exam":"upsc-cse"}' \
  "$BASE/sme/notifications/broadcast" | jq -c '{messageId: (.messageId|type), topic}'
# EXPECT: {"messageId":"string","topic":"exam_upsc-cse"}   ← hyphens preserved

curl -s -X POST -H "x-api-key: $KEY" -H 'content-type: application/json' \
  -d '{"title":"smoke","body":"smoke","exam":"nope"}' \
  "$BASE/sme/notifications/broadcast" | jq -r '.message'
# EXPECT: Unknown exam "nope". Create it via POST /sme/exams first.
```

**Segment — `exam` is echoed, validated, and the cohort predicate SURVIVES it.** This is the
regression that shipped broken; assert it numerically rather than eyeballing the send.

```bash
# 1. Unknown slug must 400 (it used to report matchedUsers: 0).
curl -s -X POST -H "x-api-key: $KEY" -H 'content-type: application/json' \
  -d '{"segment":"trial","title":"x","body":"x","exam":"nope"}' \
  "$BASE/sme/notifications/segment" | jq -r '.message'
# EXPECT: Unknown exam "nope". Create it via POST /sme/exams first.

# 2. THE ASSERTION: trial + upsc-cse must be a SUBSET of trial, not the whole free base.
#    ⚠️ There is no dry-run — /segment SENDS. Do this on STAGING
#    (https://staging-api.prepmonkey.com/api/v1), or accept two real pushes.
for E in '' ',"exam":"upsc-cse"'; do
  curl -s -X POST -H "x-api-key: $KEY" -H 'content-type: application/json' \
    -d "{\"segment\":\"trial\",\"title\":\"smoke\",\"body\":\"smoke\"$E}" \
    "$BASE/sme/notifications/segment" | jq -c '{exam, matchedUsers}'
done
curl -s -H "x-api-key: $KEY" "$BASE/sme/users?limit=1" | jq '{allUsers: .total}'
# EXPECT: {"exam":null,"matchedUsers":N} then {"exam":"upsc-cse","matchedUsers":M}
#         with M <= N, and BOTH far below allUsers.
# 🔴 If M is anywhere near allUsers, you are on the OLD image: the exam clause is still
#    replacing the trial predicate, and a "trial" push scoped to UPSC goes to the entire
#    free base. Stop sending scoped premium/trial pushes until the new image is serving.
```

**Surveys — `targetExams` round-trips, `*` is refused, and the exam breakdown appears.**

```bash
SID=<a survey id>
curl -s -X PATCH -H "x-api-key: $KEY" -H 'content-type: application/json' \
  -d '{"targetExams":["*"]}' "$BASE/sme/surveys/$SID" | jq -r '.message'
# EXPECT: targetExams does not accept "*" — leave the array empty to target every exam.

curl -s -X PATCH -H "x-api-key: $KEY" -H 'content-type: application/json' \
  -d '{"targetExams":["nope"]}' "$BASE/sme/surveys/$SID" | jq -r '.message'
# EXPECT: Unknown exam "nope". Create it via POST /sme/exams first.

curl -s -H "x-api-key: $KEY" "$BASE/sme/surveys/$SID/results?breakdown=exam" \
  | jq -c '{keys: (.results[0].breakdown.by), examNullCount, examNullBucketedAs}'
# EXPECT: {"keys":"exam","examNullCount":<n>,"examNullBucketedAs":"upsc-cse"}
# Those three keys must be ABSENT for breakdown=tier — check that too:
curl -s -H "x-api-key: $KEY" "$BASE/sme/surveys/$SID/results?breakdown=tier" \
  | jq -e 'has("examNullCount") | not'

curl -s -H "x-api-key: $KEY" "$BASE/sme/surveys/$SID/responses?limit=1" | jq -e '.data[0] | has("exam")'
curl -s -H "x-api-key: $KEY" "$BASE/sme/surveys/$SID/responses/export.csv" | head -1 | rev | cut -d, -f1 | rev
# EXPECT the CSV header to END with: exam
```

**Feedback — `?exam=` filters, `none` is the NULL bucket, and rows carry `examId`.**

```bash
curl -s -H "x-api-key: $KEY" "$BASE/sme/feedback/reports?exam=nope" | jq -r '.message'
# EXPECT: Unknown exam "nope". Create it via POST /sme/exams first.

curl -s -H "x-api-key: $KEY" "$BASE/sme/feedback/reports?limit=1" | jq -e '.data[0] | has("examId")'

# The two buckets must be DISJOINT — this is the rule that is inverted vs active_exam_id.
A=$(curl -s -H "x-api-key: $KEY" "$BASE/sme/feedback/reports?exam=upsc-cse&limit=1" | jq '.total')
B=$(curl -s -H "x-api-key: $KEY" "$BASE/sme/feedback/reports?exam=none&limit=1"     | jq '.total')
T=$(curl -s -H "x-api-key: $KEY" "$BASE/sme/feedback/reports?limit=1"               | jq '.total')
echo "$A + $B <= $T"
# EXPECT A+B ≤ T (equality only if upsc-cse and none are the only two buckets).
# 🔴 A must NOT include the NULL rows: ?exam=upsc-cse EXCLUDES unrecorded history here.
```

Finally, confirm the rollup survived the regrain — one IST day after the deploy, the daily
table should be gaining `exam_id`-tagged rows and no duplicate `(day, user_id, route)` pairs:

```sql
SELECT day, user_id, route, count(*)
FROM api_usage_daily
WHERE day >= (now() + interval '330 minutes')::date - 1
GROUP BY 1,2,3 HAVING count(*) > 1
LIMIT 5;
-- EXPECT zero rows. Any result means a NULL-exam bucket is being re-inserted every run.
```
