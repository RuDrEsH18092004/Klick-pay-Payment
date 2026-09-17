# KlickPay API Contract

## 1. API Version

KlickPay API v3, documentation version v0.6.

## 2. Base URL

### Staging

``` text
https://api.staging.klick-pay.com/v3
```

### Production

Must be confirmed by KlickPay.

## 3. Authentication

### Endpoint

``` http
POST /auth/token
```

Credentials use the documented `client-id` and `client-secret`.

Protected requests:

``` http
Authorization: Bearer <access_token>
```

KlickPay also documents:

``` http
POST /auth/renew
```

for renewal/refresh handling.

## 4. General Link

### Endpoint

``` http
POST /payments/general-link
```

Use this as the Phase 1 checkout endpoint.

Reason: General Link is the all-in-one hosted checkout and allows the
customer to choose the available payment method.

## 5. Request

Illustrative structure:

``` json
{
  "client_name": "Customer Name",
  "client_email": "customer@example.com",
  "client_phone": "+965XXXXXXXX",
  "transction_reference": "SI-00045-CP-01",
  "transction_amount": 100.00,
  "success_page": "https://erp.example.com/clickpay/success",
  "error_page": "https://erp.example.com/clickpay/error",
  "source": "web_app",
  "transaction_details": {
    "items": [
      {
        "item_code": "ITEM-001",
        "item_name": "Example Item",
        "item_quantity": 1,
        "item_unit_price": 100.00,
        "item_total_price": 100.00,
        "item_currency": "KWD"
      }
    ]
  }
}
```

Important: the gateway's field spelling is `transction_reference` and
`transction_amount`. Do not silently rename these when constructing the
actual gateway payload.

## 6. Required Fields

  Field                  Required   Rule
  ---------------------- ---------- -----------------------
  client_name            Yes        Customer name
  client_email           Yes        Valid customer email
  client_phone           Yes        Customer phone
  transction_reference   Yes        Unique, max 50 chars
  transction_amount      Yes        Decimal KWD amount
  success_page           Yes        HTTPS
  error_page             Yes        HTTPS
  source                 Yes        Recommended `web_app`
  transaction_details    No         Optional

## 7. Response

Expected data includes:

``` text
success
code
message
data.payment_id
data.invoice_token
data.payment_url
data.session_id
data.amount
data.currency
data.payment_method
request_id
```

Persist:

-   payment_id;
-   invoice_token;
-   payment_url;
-   amount;
-   currency;
-   request_id;
-   session_id if returned.

## 8. Payment URL

Do not modify `data.payment_url`.

Store it exactly as returned and provide it to the customer.

## 9. Status

### Endpoint

``` http
POST /payments/status
```

Request:

``` json
{
  "payment_id": "..."
}
```

Successful status example semantics:

``` text
event = payment.success
status = paid
result = CAPTURED
currency = KWD
```

The implementation must verify:

``` text
success == true
payment_id == local payment_id
status == paid
result == CAPTURED
amount == expected amount
currency == KWD
```

Only after these checks should accounting proceed.

## 10. Other Payment Endpoints

KlickPay documents:

``` text
POST /payments/knet
POST /payments/mpgs
POST /payments/apple-pay
POST /payments/google-pay
POST /payments/samsung-pay
POST /payments/general-link
```

Phase 1 uses General Link.

## 11. Refund APIs --- Phase 2

Full:

``` http
POST /refunds/full
```

Input:

``` json
{
  "payment_id": "..."
}
```

Partial:

``` http
POST /refunds/partial
```

Input includes:

``` json
{
  "payment_id": "...",
  "refund_amount": 25.00
}
```

## 12. Refund States

The integration should support:

``` text
success
failed
pending_collection
under_review
```

## 13. API Error Handling

For every API call:

1.  apply a finite timeout;
2.  capture HTTP status;
3.  capture gateway `code`;
4.  capture gateway `message` safely;
5.  capture `request_id`;
6.  do not expose credentials;
7.  decide whether the operation is safe to retry.

Payment creation timeout requires special care because the request may
have reached KlickPay even if ERPNext did not receive the response. Do
not blindly create multiple payments.

## 14. Retry Policy

Safe retry candidates:

-   transient status API failure;
-   temporary network failure after a known payment ID exists;
-   webhook processing failure.

Unsafe automatic retry:

-   payment creation when the original request's final outcome is
    unknown.

For unknown creation outcomes, reconcile using gateway
identifiers/status where available before creating another payment.