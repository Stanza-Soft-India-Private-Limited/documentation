# Razorpay Integration - Installation & Setup Guide

## Step 1: Install Dependencies

You need to install the Razorpay Node.js SDK:

```bash
npm install razorpay
npm install --save-dev @types/razorpay
```

## Step 2: Configure Environment Variables

Add the following to your `.env` file:

```bash
# Razorpay Configuration
RAZORPAY_KEY_ID=rzp_live_YOUR_KEY_ID_HERE
RAZORPAY_KEY_SECRET=YOUR_KEY_SECRET_HERE
RAZORPAY_WEBHOOK_SECRET=YOUR_WEBHOOK_SECRET_HERE
RAZORPAY_MODE=live
COURSE_PRICE=9900
```

**Replace the placeholder values with your actual Razorpay credentials.**

See `docs/RAZORPAY_ENV_VARS.md` for detailed instructions on getting these values.

## Step 3: Run Database Migration

Generate Prisma client and apply migration:

```bash
npm run db:migrate
```

When prompted for migration name, enter: `add_payment_models`

This will:
- Create Order, Payment, and WebhookEvent tables
- Add OrderStatus and PaymentStatus enums
- Generate updated Prisma client

## Step 4: Configure Razorpay Dashboard

### 4.1 Enable Payment Methods
1. Login to Razorpay Dashboard
2. Go to Settings → Payment Methods
3. Enable desired payment methods (UPI, Cards, NetBanking, Wallets)

### 4.2 Setup Webhook
1. Go to Settings → Webhooks
2. Click "Add New Webhook"
3. Configure:
   - **URL**: `https://app.stanzasoft.ai/webhooks/razorpay`
   - **Secret**: Generate and copy (add to `.env` as `RAZORPAY_WEBHOOK_SECRET`)
   - **Events**: Select:
     - ✅ `payment.authorized`
     - ✅ `payment.captured`
     - ✅ `payment.failed`
     - ✅ `order.paid`
4. Click "Create Webhook"

## Step 5: Verify Installation

Start your development server:

```bash
npm run start:dev
```

Check the logs for:
```
Razorpay gateway initialized successfully
```

If you see this, the integration is ready!

## Step 6: Test the Integration

### 6.1 Get JWT Token

First, authenticate and get a JWT token:

```bash
curl -X POST https://app.stanzasoft.ai/api/v1/auth/cognito/exchange \
  -H "Content-Type: application/json" \
  -d '{
    "idToken": "YOUR_COGNITO_ID_TOKEN",
    "accessToken": "YOUR_COGNITO_ACCESS_TOKEN",
    "refreshToken": "YOUR_COGNITO_REFRESH_TOKEN",
    "provider": "google"
  }'
```

Save the `accessToken` from the response.

### 6.2 Create Order

```bash
curl -X POST https://app.stanzasoft.ai/api/v1/payments/orders \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "metadata": {
      "course": "UPSC CSE 2025",
      "module": "Prelims"
    }
  }'
```

Expected response:
```json
{
  "success": true,
  "message": "Order created successfully",
  "orderId": "uuid-here",
  "razorpayOrderId": "order_MhXUt9xOVBjL9a",
  "amount": 9900,
  "currency": "INR",
  "keyId": "rzp_live_xxxxx"
}
```

### 6.3 Client-Side Payment (Android/iOS)

Use the `razorpayOrderId` and `keyId` from the response to initiate payment on the client side using Razorpay SDK.

After successful payment, the client will receive:
- `razorpay_order_id`
- `razorpay_payment_id`
- `razorpay_signature`

### 6.4 Verify Payment

```bash
curl -X POST https://app.stanzasoft.ai/api/v1/payments/verify \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "razorpay_order_id": "order_MhXUt9xOVBjL9a",
    "razorpay_payment_id": "pay_MhXUt9xOVBjL9b",
    "razorpay_signature": "signature_from_client"
  }'
```

Expected response:
```json
{
  "success": true,
  "message": "Payment verified successfully",
  "paymentId": "uuid-here",
  "orderId": "uuid-here",
  "status": "CAPTURED",
  "method": "upi",
  "amount": 9900
}
```

### 6.5 Get Payment History

```bash
curl -X GET https://app.stanzasoft.ai/api/v1/payments/history \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

Expected response:
```json
{
  "success": true,
  "total": 1,
  "orders": [
    {
      "orderId": "uuid-here",
      "razorpayOrderId": "order_MhXUt9xOVBjL9a",
      "amount": 9900,
      "currency": "INR",
      "status": "PAID",
      "createdAt": "2025-12-05T08:00:00.000Z",
      "payment": {
        "paymentId": "uuid-here",
        "razorpayPaymentId": "pay_MhXUt9xOVBjL9b",
        "status": "CAPTURED",
        "method": "upi",
        "capturedAt": "2025-12-05T08:01:00.000Z"
      }
    }
  ]
}
```

## Step 7: Test Webhooks

### Option 1: Use Razorpay Dashboard
1. Go to Settings → Webhooks
2. Click on your webhook
3. Click "Send Test Webhook"
4. Select event type (e.g., `payment.captured`)
5. Check your server logs for webhook processing

### Option 2: Use ngrok (for local testing)
```bash
# Install ngrok
npm install -g ngrok

# Start ngrok tunnel
ngrok http 3000

# Update webhook URL in Razorpay Dashboard to ngrok URL
# e.g., https://abc123.ngrok.io/webhooks/razorpay
```

## Troubleshooting

### Issue: "Razorpay credentials missing"
- Check `.env` file has correct values
- Restart server after updating `.env`

### Issue: "Invalid webhook signature"
- Verify `RAZORPAY_WEBHOOK_SECRET` matches Razorpay Dashboard
- Check webhook secret is for correct mode (test/live)

### Issue: "Order not found"
- Ensure order was created successfully
- Check database for order record
- Verify `razorpay_order_id` is correct

### Issue: "Payment verification failed"
- Check signature is correct from client
- Verify using test mode credentials if testing
- Check Razorpay dashboard for payment status

## Security Checklist

- ✅ Never commit `.env` file to version control
- ✅ Use different credentials for test and live mode
- ✅ Webhook signature validation is enabled
- ✅ Rate limiting is configured (5 req/min for order creation)
- ✅ JWT authentication required for all payment endpoints
- ✅ Amount validation on server side (not trusting client)

## Next Steps

1. **Update Course Price**: Change `COURSE_PRICE` in `.env` when needed
2. **Add More Payment Features**: Refunds, partial payments, etc.
3. **Monitor Payments**: Check Razorpay Dashboard regularly
4. **Setup Alerts**: Configure email/SMS alerts for failed payments
5. **Analytics**: Track conversion rates and payment methods

## API Endpoints Summary

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/v1/payments/orders` | POST | Required | Create order |
| `/api/v1/payments/verify` | POST | Required | Verify payment |
| `/api/v1/payments/history` | GET | Required | Get payment history |
| `/webhooks/razorpay` | POST | Webhook Signature | Handle webhooks |

## Support

For Razorpay-specific issues:
- Documentation: https://razorpay.com/docs/
- Support: https://razorpay.com/support/

For integration issues:
- Check server logs
- Verify environment variables
- Test with Razorpay test mode first
