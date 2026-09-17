# KlickPay Webhook Implementation

## 1. Endpoint

Create a public HTTPS endpoint dedicated to KlickPay.

Example:

``` text
https://erp.example.com/api/method/clickpay_integration.webhook.handle
```

The exact public route can be chosen during implementation.

## 2. Required Headers

``` text
Content-Type: application/json
X-KlickPay-Event
X-KlickPay-Timestamp
X-KlickPay-Signature
```

## 3. Critical Rule: Raw Body

The signature must be calculated over the exact raw request body.

Correct:

``` text
request body bytes
        ↓
HMAC-SHA256(secret, raw_body)
        ↓
hex digest
        ↓
constant-time compare
```

Do not:

``` text
parse JSON
→ reserialize JSON
→ calculate signature
```

because formatting/order changes can invalidate the signature.

## 4. Verification Algorithm

Pseudo-code:

``` python
raw_body = request.get_data()

timestamp = request.headers.get("X-KlickPay-Timestamp")
signature = request.headers.get("X-KlickPay-Signature")

validate_required_headers()

validate_timestamp(timestamp)

expected = hmac_sha256(
    secret=webhook_secret,
    message=raw_body
).hexdigest()

if not constant_time_compare(expected, signature):
    reject()

payload = json.loads(raw_body)
```

## 5. Replay Protection

Reject old webhook timestamps.

Baseline implementation:

``` text
5-minute tolerance
```

This must be confirmed against KlickPay's final production
recommendation.

Do not accept a webhook simply because the HMAC is valid if its
timestamp is outside the allowed window.

## 6. Idempotency

For payment events:

``` text
idempotency_key = event_type + ":" + payment_id
```

For refund events:

``` text
idempotency_key = event_type + ":" + refund_reference
```

Store the key in `ClickPay Webhook Log`.

If already processed:

-   mark event as Duplicate;
-   do not create another Payment Entry;
-   return an appropriate successful acknowledgement if the event was
    already safely handled.

## 7. Supported Events

``` text
payment.success
payment.declined
payment.failed
payment.voided

refund.success
refund.failed
refund.pending_balance
refund.under_review
```

Refund events are Phase 2.

## 8. Payment Success Processing

``` text
payment.success
    ↓
Find ClickPay Transaction
    ↓
Validate payment_id
    ↓
Validate amount/currency
    ↓
Call /payments/status
    ↓
Require paid + CAPTURED
    ↓
Check idempotency
    ↓
Create Payment Entry
    ↓
Submit Payment Entry
    ↓
Mark transaction Paid
```

## 9. Event Ordering

Any order is valid:

``` text
Webhook → Redirect
```

or:

``` text
Redirect → Webhook
```

Accounting must not depend on redirect order.

## 10. Unknown Payment ID

If webhook references an unknown payment ID:

-   do not create accounting;
-   persist the webhook;
-   mark it as an exception;
-   investigate/reconcile.

Do not create a new ERPNext transaction from untrusted webhook data.

## 11. Amount Mismatch

Example:

``` text
Local transaction = 50 KWD
Gateway webhook/status = 60 KWD
```

Do not create Payment Entry automatically.

Mark:

``` text
Exception / Amount Mismatch
```

and alert support/finance.

## 12. Currency Mismatch

The implementation is KWD-focused.

If gateway status is not KWD:

-   block automatic accounting;
-   log mismatch;
-   raise exception.

## 13. Fast Acknowledgement

Webhook handler should:

1.  authenticate;
2.  validate;
3.  persist;
4.  queue;
5.  return 2xx.

Do not perform slow accounting/API operations synchronously if it risks
gateway timeout.

## 14. Security Logging

Log:

-   event type;
-   payment/refund reference;
-   timestamp;
-   signature valid/invalid;
-   processing result.

Never log:

-   webhook secret;
-   bearer token;
-   client secret.
