# Payments Rebuild — Rollout, Portal Config & Live-Test Checklist

This is the "sponge wall" guard: even with all the code shipped, the webhook-as-
source-of-truth + reconciliation only work if the **provider dashboards** point at
our endpoints with the right secret + events. Verify every box below.

Prod base URL: `https://app.stanzasoft.ai/api/v1`
Probe done 2026-06-18: `/health` 200, `/webhooks/razorpay` 401 (route live, guard active),
`/webhooks/apple` 400 (route live). So the endpoints are reachable — the open question
is the dashboard configuration below.

---

## 1. Razorpay dashboard (Settings → Webhooks)
- [ ] **Webhook URL** = `https://app.stanzasoft.ai/api/v1/webhooks/razorpay`
      🚨 **FOUND WRONG 2026-06-18:** dashboard was set to `…/webhooks/razorpay`
      (NO `/api/v1`) → **404 → every Razorpay webhook silently dropped** (major cause
      of "paid but pending"). The global prefix `api/v1` only excludes `config`, so
      webhooks REQUIRE the prefix. Apple's URLs already include it. **Edit the
      Razorpay URL to add `/api/v1/`.**
- [ ] **Status = Active.** ⚠️ Razorpay AUTO-DISABLES a webhook after ~24h of failures.
      The OLD code verified the signature over a re-serialized body → every webhook
      likely failed → it may already be **disabled**. If so, **re-enable it** (this
      alone may explain the stuck Razorpay orders).
- [ ] **Secret** matches the backend env `RAZORPAY_WEBHOOK_SECRET` EXACTLY (the new
      raw-body HMAC uses this secret; a mismatch = every webhook 401s).
- [ ] **Active events:** `payment.captured`, `order.paid`, `payment.failed`,
      `refund.created`, `refund.processed`, `refund.failed`,
      `payment.dispute.created`, `payment.dispute.won`, `payment.dispute.lost`,
      `payment.dispute.closed`
- [ ] Configure the webhook in BOTH **Live** and **Test** mode (test-mode webhook
      needed for the test-mode live-test below).
- **Self-verify:** dashboard → Webhooks → "Send test webhook" → expect HTTP 200 +
  a row appears in the `payment_events` table.

## 2. Apple App Store Connect
- [ ] App → **App Information → App Store Server Notifications (Version 2)**:
  - [ ] **Production URL** = `https://app.stanzasoft.ai/api/v1/webhooks/apple`
  - [ ] **Sandbox URL** = `https://app.stanzasoft.ai/api/v1/webhooks/apple` (same)
  - ⚠️ If these were never set, Apple never told us about purchases/renewals — a
    prime suspect for "the ₹5000 isn't in the backend."
- [ ] **App Store Server API key** (Users & Access → Integrations → In-App Purchase →
      generate a key) — needed by the reconciliation/recovery client:
  - `APPLE_KEY_ID`, `APPLE_ISSUER_ID`, `APPLE_PRIVATE_KEY_BASE64` set in backend env.
- **Self-verify the Apple webhook end-to-end (no purchase needed)** — after deploy +
  URL configured, ask Apple to send a TEST notification and poll the result:
  ```sh
  KEY="<API_KEY_SECRET>"; BASE="https://app.stanzasoft.ai/api/v1"
  TOKEN=$(curl -s -X POST -H "x-api-key: $KEY" $BASE/payments/apple/test-notification | jq -r .token)
  # backend logs should show: "Apple notification: apple.TEST" + a payment_events row
  curl -s -H "x-api-key: $KEY" "$BASE/payments/apple/test-notification/$TOKEN" | jq
  # → delivered:true (sendAttemptResult SUCCESS) proves URL + JWS verify + 2xx work
  ```
  `delivered:false` or a non-SUCCESS send attempt = the ASC URL/config is wrong → fix it.

## 3. Backend env (mau runtime) — confirm present
- [ ] `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET`, `RAZORPAY_MODE=live`
- [ ] `APPLE_BUNDLE_ID`, `APPLE_APP_ID`, `APPLE_ISSUER_ID`, `APPLE_KEY_ID`, `APPLE_PRIVATE_KEY_BASE64`, `APPLE_ENVIRONMENT=Production`
- [ ] `API_KEY_SECRET` (recovery / notifications / SME)
- [ ] `PAYMENT_RECONCILE_ENABLED` — leave unset/true normally; set `false` only while
      reviewing the recovery dry-run, then re-enable.

---

## 4. Deploy steps (in order)
1. [ ] Apply migration `prisma/migrations/20260618000000_payments_rebuild/migration.sql`.
2. [ ] Deploy backend (`mau`). Confirm new revision serving: `/api/v1/payments/order/test/status`
       returns **401** (auth required), NOT 404 (404 = old code still serving).
3. [ ] Confirm boot logs: "Payment reconcile scheduled (7 jobs)". A startup throw
       "Apple root certificates missing or empty" means the certs aren't in the image — fix before relying on Apple.
4. [ ] **Recovery (dry-run first):** with `PAYMENT_RECONCILE_ENABLED=false`,
       `ts-node -r tsconfig-paths/register scripts/recover-payments.ts` → review the
       JSON (Razorpay stuck orders + Apple subs it would grant) → re-run with
       `--execute` → re-enable reconcile. (The ₹5000, having NO local record, won't
       appear here — it heals when that user reopens the iOS app, or via an SME
       manual grant after an App Store Connect lookup.)

---

## 5. Live-test matrix (after deploy + new mobile build)
**Razorpay (test mode):**
- [ ] Full success → lands on Success screen, premium reflects without restart.
- [ ] UPI app-switch + kill app mid-payment → reopen → resumes → grants.
- [ ] Force-close the result before verify → Profile "Refresh status" → grants.
- [ ] Refund a test payment → user downgraded.
- [ ] Re-run a verify (retry) → success, not "verification failed".

**Apple (Sandbox):**
- [ ] Purchase → Success screen; kill before verify → relaunch → entitlement reconciled + granted.
- [ ] Restore Purchases works.
- [ ] Renewal / Expiry / Refund via sandbox → entitlement updates (check `payment_events` + `apple_subscriptions`).

**Both:** every captured payment leaves a `payment_events` row; the divergence-alert
cron logs `[PAYMENT DIVERGENCE]` if anything is captured-but-not-granted.
