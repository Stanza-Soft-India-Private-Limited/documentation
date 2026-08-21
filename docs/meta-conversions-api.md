# Meta Conversions API (server-side purchase events)

| | |
|---|---|
| Purpose | Report `Subscribe` + `Purchase` to Meta Events Manager (app `4453080351629453`) straight from the backend, at the moment a payment is verified — independent of the client SDK, ATT opt-in, and iOS's StoreKit 2 auto-logger |
| Implementation | `src/common/meta/meta-conversions.service.ts`, `src/common/meta/meta-conversions.module.ts` |
| Config | `src/config/meta.config.ts` |
| Call sites | `PaymentsService.verifyPayment` (Razorpay), `ApplePaymentsService.verifyAppleTransaction` + `processSubscriptionEvent` (Apple client-verify and App Store Server Notifications) |
| Tests | None added — this integration ships with no new tests per the task's hard rule; the existing `payments.service.spec.ts` / `apple-payments.service.spec.ts` continue to pass unmodified with the dependency left undefined (see "Testability" below) |

---

## 1. Why this exists

Meta Events Manager only ever heard from the mobile client SDK (`MetaAnalytics.kt`
→ FBSDKCoreKit / Android AppEventsLogger). That path is structurally lossy:

- iOS "ATE True Status Rate" (ATT opt-in) measured **17%** — the SDK cannot log
  a matchable event for the other 83%.
- FBSDKCoreKit 18.x's StoreKit 2 auto-logger was verified, live, to emit **zero**
  Purchase/Subscribe events for this app across a 4-week window with confirmed
  real purchases (2026-08-21) — the assumption it "just works" for iOS was false.

A server-side Conversions API path fixes both: it fires once a payment is
**verified by our own backend**, so it can never report an unpaid order, needs
no device SDK, and is unaffected by ATT.

## 2. The API contract implemented

**Endpoint:** `POST https://graph.facebook.com/{API_VERSION}/{DATASET_ID}/events?access_token={TOKEN}`

Docs consulted (fetched 2026-08-21 via WebFetch, quoted in the service's
header comment):

- Conversions API for App Events — <https://developers.facebook.com/documentation/ads-commerce/conversions-api/app-events>
- Conversions API overview — <https://developers.facebook.com/docs/marketing-api/conversions-api/>
- Customer information parameters (em/ph hashing rules) — <https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/customer-information-parameters>
- `extinfo` field order — <https://developers.facebook.com/docs/graph-api/reference/application/activities#parameters>

Request body shape:

```json
{
  "data": [
    {
      "event_name": "Purchase",
      "event_time": 1755765461,
      "event_id": "razorpay_pay_ABC123",
      "action_source": "app",
      "user_data": {
        "em": ["<sha256 hex>"],
        "ph": ["<sha256 hex>"],
        "external_id": ["<sha256 hex>"]
      },
      "custom_data": {
        "currency": "INR",
        "value": "599.00",
        "content_ids": ["com.stanzasoft.upscbuddy.premium.monthly"],
        "content_type": "subscription"
      },
      "app_data": {
        "advertiser_tracking_enabled": 0,
        "extinfo": ["a2", "com.stanzasoft.upscbuddy", "", "", "", "", "", "", "", "", "", "", "", "", "", ""]
      }
    }
  ],
  "test_event_code": "TEST12345"
}
```

`test_event_code` is only present when `META_TEST_EVENT_CODE` is set.

### Fields we deliberately do NOT send

- `madid` (device advertising id) — not available server-side.
- `client_ip_address` — the request that triggers this isn't the purchasing
  device's own live request in every case (e.g. an App Store Server
  Notification arrives from Apple's servers, not the user's phone), so sending
  it would sometimes be wrong. Omitted entirely rather than sent wrong.
- `application_tracking_enabled`, `campaign_ids`, `install_referrer` — no
  install-attribution signal is available at payment-verification time.

`app_data.advertiser_tracking_enabled` is always sent as `0`. We have no ATT
signal at all in this context (this fires from a payment-verification call,
not a device SDK session) and send no `madid` — `0` is the honest value; we
never claim tracking we can't prove.

## 3. Env vars an operator must set

None of these exist in `.env` yet — this integration is genuinely new and
starts fully inert.

| Var | Required | Notes |
|---|---|---|
| `META_APP_ID` | Yes (for a working setup) | Meta App ID, `4453080351629453`. Also the default `META_DATASET_ID`. |
| `META_DATASET_ID` | No — falls back to `META_APP_ID` | Only set if this app's dataset id differs from its App ID (Business Manager shared dataset). |
| `META_ACCESS_TOKEN` | **Yes** | System-user or app access token with the dataset's `ads_management`/`business_management` permission. Generate in Events Manager → Settings → Conversions API → "Generate access token". Never log or commit this. |
| `META_API_VERSION` | No, default `v21.0` | Bump when Meta deprecates a version — env change, no deploy of code logic needed. |
| `META_TEST_EVENT_CODE` | No | Set temporarily while verifying in Events Manager's Test Events tab (Settings → Test Events); unset for production traffic. |
| `META_APP_PACKAGE_NAME` | No, default `com.stanzasoft.upscbuddy` | Stamped into `app_data.extinfo[1]`. Matches both platforms' real package/bundle id today. |

**Only `META_DATASET_ID` (or its `META_APP_ID` fallback) + `META_ACCESS_TOKEN`
gate whether the integration is "configured".** `META_APP_ID` alone with no
token still leaves it inert.

## 4. Proof it is inert when unconfigured

`MetaConversionsService` computes `configured = Boolean(datasetId && accessToken)`
once in its constructor:

- `onModuleInit()` logs **exactly one** `logger.warn(...)` at boot when
  unconfigured, explaining what's missing and that this is safe.
- `sendPurchaseSignal()` — the only method that makes an HTTP call — returns
  immediately (`if (!this.configured) return;`) with **no further log, no
  HTTP call, no throw**, on every subsequent invocation.
- Nothing in `PaymentsService` or `ApplePaymentsService` awaits the result in
  a way that could fail the request — both call sites use `void
  this.fireMetaPurchaseSignals(...)`, and `MetaConversionsService` itself
  never rejects (every code path inside `sendPurchaseSignal` is wrapped in
  try/catch that logs and swallows, never rethrows).

Verified: `.env` in this repo has no `META_*` vars today (confirmed via `rg`
before writing any code), `npx tsc --noEmit` and `npx nest build` both pass
clean, and the existing payment test suites (112 tests, 4 suites) pass
unmodified — see "Testability" below for why that's a meaningful check of the
inert path, not just an unrelated pass.

## 5. Deduplication against the client SDK

Meta's app-events dedup is **`event_id` + `event_name`** based (both quoted
directly from the App Events CAPI doc): *"The logic leverages the field
`event_id` and `event_name` based deduplication. Conversions API and SDK / App
Events API events that carry the same `event_id`"* are collapsed to one.

**The shared id is never invented — it's an id the payment provider already
hands to both sides independently:**

| Provider | Event id | Client has it via | Server has it via |
|---|---|---|---|
| Razorpay | `razorpay_<razorpayPaymentId>` | Razorpay Checkout SDK's success callback (`razorpay_payment_id`), before the app even calls our `/verify` endpoint | The `razorpayPaymentId` parameter of `verifyPayment()` |
| Apple | `apple_<transactionId>` | StoreKit 2's `Transaction.id` on the purchased/updated transaction (`success.transactionId` in `PaymentPlatform.kt`) | `decoded.transactionId` from the verified JWS |

`MetaConversionsService.buildEventId(provider, id)` is the single place this
string is constructed, so both call sites are guaranteed to agree with each
other.

### ⚠️ This only actually dedups once the mobile client sends `event_id` too

**As of this change, it does not.** I read `MetaAnalytics.kt` and
`PaymentViewModel.kt` in the mobile repo (read-only — mobile is out of scope
for this task, a parallel agent owns it) and confirmed:

- `MetaAnalytics.logSubscribe()` / `logPurchase()` take `(amountMajor,
  currencyIso4217, contentId)` — there is no `event_id` parameter today.
- Meta's own iOS example for attaching a dedup id is a raw parameter:
  `AppEvents.shared.logEvent(.achievedLevel, parameters:
  [AppEvents.ParameterName(rawValue: "event_id"): "123"])`. The generic
  `logEvent`/`logEventWithValue` path can carry this today by adding
  `"event_id"` to the params map; whether the Android/iOS `logPurchase`
  convenience methods used for the `Purchase` event specifically accept a
  custom `event_id` parameter the same way needs to be verified against the
  installed FBSDK version — I did not verify this, since it's a mobile-side
  change outside this task's scope.

**Until the mobile side is wired to send `event_id = razorpay_<paymentId>` /
`apple_<transactionId>` on its `Subscribe`/`Purchase` calls, every purchase
will be double-counted in Events Manager** (once from the client SDK, once
from this backend path). This is flagged here deliberately rather than
silently shipped as "solved" — the backend side is correct and ready, but
dedup is a two-sided contract and only one side is in this change.

**Until dedup is completed, consider using `META_TEST_EVENT_CODE`** to route
this backend's events to Events Manager's Test Events tab (separate from
production Purchase/Subscribe counts) rather than letting them count
production conversions twice.

## 6. Amount reported

Always the amount **actually charged**, never a plan's list price:

- **Razorpay:** `paymentDetails.amount` from `RazorpayGateway.fetchPayment()`
  — the real captured amount in paise, cross-checked against our order before
  this ever fires (see `verifyPayment` Step 4's amount/currency mismatch
  guards). A campaign buyer charged ₹4,999 against a ₹5,900 list price is
  reported as ₹4,999.
- **Apple:** `decoded.price` (MILLIUNITS) / `decoded.currency` from the
  verified JWS, via the *same* `chargedAmountForResponse()` helper the
  client-facing verify response already returns as `chargedAmountMajor` /
  `chargedCurrency` — server and client report an identical value by
  construction, not by coincidence. When Apple didn't supply a price (pre-2023
  transactions), the Meta event is **skipped, not guessed** — see
  `ApplePaymentsService.fireMetaPurchaseSignals`'s doc comment.

## 7. Where each event fires

Both events (`Subscribe` then `Purchase`, same as the client — mirrors
`onRazorpayGranted` / `logAppleMetaConversionOnce` in `PaymentViewModel.kt`)
fire from:

1. **`PaymentsService.verifyPayment`** — only inside the `if (isCaptured)`
   branch, after `grantPremiumForOrder`. A retried `verifyPayment` call for an
   already-granted order short-circuits earlier (Step 3's idempotent check)
   and never re-reaches this code.
2. **`ApplePaymentsService.verifyAppleTransaction`** — only on a genuinely new
   `appleTransactionId` (a repeat verify of the same transaction hits the
   `P2002`/`isExisting` branch and returns before this fires). Skipped when
   `isTest` (Sandbox/TestFlight) — those aren't real revenue and must not
   pollute the ad-optimization signal.
3. **`ApplePaymentsService.processSubscriptionEvent`** — the shared handler
   behind `SUBSCRIBED`, `OFFER_REDEEMED`, `DID_RENEW`, `REFUND_REVERSED` App
   Store Server Notifications. Fires only when this call minted a genuinely
   new order/payment row (tracked via `isNewCharge`, not merely "handled a
   notification") and, again, never for `isTest`. `DID_RENEW` legitimately
   reaches here — Apple mints a new `transactionId` per renewal, which is a
   real new charge worth reporting, independent of whether the in-app
   "Premium activated" push fires (`opts.notify`).

No event fires from `handleExpiryOrRevoke`, `DID_FAIL_TO_RENEW`, `EXPIRED`,
`REVOKE`, `REFUND`, `createManualPremiumGrant` (SME comps), or any
reconciliation/webhook path that isn't a genuine new charge.

## 8. Failure containment

- `MetaConversionsService.sendPurchaseSignal()` never throws — every failure
  (HTTP error, timeout, anything) is caught and logged via
  `this.logger.error(...)` with the HTTP status and message, **never
  swallowed silently** (this project has a hard rule against exactly that).
  Raw response bodies and PII are never logged — only status code + message.
- Every HTTP call carries a 5-second timeout
  (`MetaConversionsService.REQUEST_TIMEOUT_MS`, enforced per-request via
  axios `timeout`, and the module's own `HttpModule.register({ timeout: 5000
  })` isolates it from the rest of the app's socket pool — same pattern as
  `WyltoModule`).
- Both call sites use `void this.fireMetaPurchaseSignals(...)` — never
  awaited by the request/webhook handler, so a slow or failing Meta call
  cannot delay or fail a payment response or a webhook 200.
- The one thing that CAN throw inside `fireMetaPurchaseSignals` (the
  `userAuth` lookup for Advanced Matching) is wrapped in its own try/catch
  that logs and falls back to sending the event with no PII rather than
  losing the conversion signal entirely.

## 9. PII hashing

Per the Customer Information Parameters doc:

- **Email (`em`)**: trim, lowercase, SHA-256 hex.
- **Phone (`ph`)**: strip everything but digits (removes the `+` from our
  stored E.164 format and any other symbols/letters), strip leading zeros,
  SHA-256 hex. Country code is preserved (E.164 already includes it), matching
  the doc's *"Phone numbers must include a country code to be used for
  matching."*
- **`external_id`**: our internal `userId` (UUID), hashed with the same
  SHA-256 even though the doc only "recommends" hashing it — no internal
  identifier is sent in the clear.
- A field is **omitted entirely, never sent unhashed or malformed**, when we
  don't have it (e.g. no phone on file) or aren't confident the normalization
  is right — see `MetaConversionsService.buildUserData()`.
- Raw PII is never logged anywhere in this integration; log lines reference
  event ids, statuses, and transaction ids only.

## 10. Testability

`MetaConversionsService` is injected as an **optional** constructor parameter
(`metaConversions?: MetaConversionsService`) in both `PaymentsService` and
`ApplePaymentsService` — not because the dependency is optional at runtime
(the real app always provides it via `MetaConversionsModule`, imported into
`PaymentsModule`), but so the existing hand-built `new PaymentsService(...)` /
`new ApplePaymentsService(...)` calls in `payments.service.spec.ts` /
`apple-payments.service.spec.ts` keep compiling and passing without editing
either spec file (per the hard rule against editing tests to make them pass).
All call sites use `this.metaConversions?.sendPurchaseSignal(...)`, so the
tests exercise the exact same inert-when-absent code path a misconfigured
production deploy would hit.
