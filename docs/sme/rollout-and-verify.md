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
