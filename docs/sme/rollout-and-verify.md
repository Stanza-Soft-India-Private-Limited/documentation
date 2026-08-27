# SME Management — Rollout & Verify Checklist

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
- `DELETE /sme/users/:id/premium` → back to `UNSUBSCRIBED`, Neo4j tier free.
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
