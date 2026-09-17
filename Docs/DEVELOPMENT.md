# KlickPay ERPNext Integration --- Development Specification

## 1. Technical Scope

Build a dedicated Frappe application that integrates ERPNext with
KlickPay API v3.

The application should contain:

-   configuration;
-   gateway API client;
-   payment-link service;
-   transaction state machine;
-   webhook endpoint;
-   signature verification;
-   status verification;
-   idempotent accounting;
-   Payment Entry creation;
-   email notification;
-   expiry processing;
-   operational recovery;
-   future refund support.

## 2. Recommended App Structure

``` text
clickpay_integration/
├── clickpay_integration/
│   ├── hooks.py
│   ├── api.py
│   ├── integrations/
│   │   └── klickpay/
│   │       ├── __init__.py
│   │       ├── client.py
│   │       ├── auth.py
│   │       ├── payments.py
│   │       ├── status.py
│   │       └── webhook.py
│   ├── services/
│   │   ├── payment_service.py
│   │   ├── accounting.py
│   │   ├── notifications.py
│   │   ├── idempotency.py
│   │   └── expiry.py
│   ├── doctype/
│   │   ├── clickpay_settings/
│   │   ├── clickpay_transaction/
│   │   ├── clickpay_webhook_log/
│   │   └── clickpay_refund/
│   └── www/
│       └── clickpay/
│           ├── success.py
│           ├── success.html
│           ├── error.py
│           └── error.html
├── tests/
│   ├── test_payment_link.py
│   ├── test_webhook.py
│   ├── test_accounting.py
│   ├── test_idempotency.py
│   └── test_expiry.py
└── README.md
```

## 3. Layer Responsibilities

### `client.py`

Only handles HTTP communication with KlickPay.

It must not contain ERPNext business logic.

Responsibilities:

-   authentication;
-   authorization header;
-   request timeout;
-   JSON serialization;
-   response parsing;
-   safe error handling;
-   request ID capture.

### `payment_service.py`

Owns business logic:

-   invoice validation;
-   amount validation;
-   transaction creation;
-   link reuse;
-   link generation;
-   status transition;
-   background processing.

### `accounting.py`

Owns ERPNext accounting:

-   validate Sales Invoice;
-   calculate allowed allocation;
-   find existing Payment Entry;
-   create Receive Payment Entry;
-   add reference to Sales Invoice;
-   set ClickPay Clearing Account;
-   submit safely.

### `webhook.py`

Owns:

-   raw request retrieval;
-   signature validation;
-   timestamp validation;
-   event validation;
-   webhook persistence;
-   idempotency key generation;
-   queueing.

### `notifications.py`

Owns:

-   email;
-   template variables;
-   resend behavior.

The notification interface should later support WhatsApp without
changing payment logic.

## 4. Server API Methods

Recommended whitelisted methods:

``` text
clickpay_integration.api.generate_payment_link
clickpay_integration.api.send_payment_link
clickpay_integration.api.check_payment_status
clickpay_integration.api.generate_new_payment_link
```

Never put gateway secrets in method arguments from the browser.

## 5. Generate Payment Link Flow

Pseudo-flow:

``` text
generate_payment_link(sales_invoice, amount=None)
    ↓
load Sales Invoice
    ↓
validate submitted / not cancelled
    ↓
calculate current outstanding
    ↓
validate requested amount
    ↓
load ClickPay Settings
    ↓
find active unexpired transaction
    ↓
if reusable:
    return existing payment_url
    ↓
authenticate
    ↓
create unique merchant reference
    ↓
POST /payments/general-link
    ↓
validate API response
    ↓
create ClickPay Transaction
    ↓
calculate expires_at
    ↓
save transaction
    ↓
return safe result to UI
```

## 6. Amount Rules

For a standard invoice:

``` text
allowed_amount <= current outstanding amount
```

For example:

``` text
Invoice = 100 KWD
Outstanding = 100 KWD

Full link = 100 KWD
Partial link = 50 KWD
```

After a confirmed 50 KWD payment:

``` text
Invoice = 100 KWD
Paid = 50 KWD
Outstanding = 50 KWD
```

The next payment must be validated against the new outstanding amount.

Do not trust a stale amount supplied by the browser.

## 7. Merchant Reference

Generate a unique reference that:

-   is no longer than 50 characters;
-   identifies the ERPNext payment attempt;
-   is safe for the gateway;
-   cannot collide.

Example:

``` text
SI-00045-CP-01
```

If the invoice number can exceed gateway limits, use a compact
deterministic/reference ID rather than truncating blindly.

## 8. Payment Transaction Creation

Persist the local transaction only after the gateway returns enough
information to identify the external payment.

Minimum successful response data:

-   payment_id;
-   payment_url;
-   amount;
-   currency;
-   request_id;
-   invoice_token where available.

The gateway URL must be stored exactly as returned.

## 9. Transaction State Machine

``` text
Created
  ↓
Link Created
  ↓
Sent
  ↓
Pending
  ├──→ Paid
  ├──→ Failed
  ├──→ Declined
  └──→ Voided

Link Created/Sent
  └──→ Expired
```

A transaction marked Paid must not return to Pending.

A terminal transaction must not be overwritten by a later stale event.

## 10. Browser Redirect

Success/error routes are customer-facing only.

The redirect handler should:

1.  read safe query parameters;
2.  locate the transaction;
3.  optionally trigger/check server-side status;
4.  display current verified state;
5.  never directly create a Payment Entry from query parameters.

Recommended customer messages:

-   `Payment successful`
-   `Payment is being verified`
-   `Payment unsuccessful`
-   `Payment link expired`
-   `Unable to verify payment; please contact support`

## 11. Background Processing

Webhook requests must be lightweight.

Recommended:

``` text
Webhook request
    ↓
Verify signature
    ↓
Validate timestamp
    ↓
Persist event
    ↓
Detect duplicate
    ↓
Queue background job
    ↓
Return 2xx
```

Worker:

``` text
Read webhook
    ↓
Find ClickPay Transaction
    ↓
Call /payments/status
    ↓
Validate gateway response
    ↓
Update transaction
    ↓
Create Payment Entry if successful
    ↓
Commit
```

## 12. Hooks

Avoid creating accounting entries directly from broad Sales Invoice
hooks.

The integration should primarily react to:

-   explicit user action for link generation;
-   webhook/background processing for payment confirmation;
-   scheduled job for expiry/recovery.

A document hook may be used for validation or UI preparation, but
payment accounting must remain gateway-confirmation-driven.

## 13. Concurrency

Payment processing must be safe when two workers process the same event.

Before Payment Entry creation:

1.  lock/re-check transaction where appropriate;
2.  check `payment_entry` field;
3.  search for existing Payment Entry containing the same gateway
    payment ID/reference;
4.  only one worker may proceed;
5.  commit the accounting transaction;
6.  update transaction with Payment Entry link.

Do not rely only on application-level `if` statements.

## 14. Payment Entry Creation

Recommended values:

``` text
Payment Type = Receive
Party Type = Customer
Party = Sales Invoice customer
Paid Amount = confirmed gateway amount
Received Amount = confirmed gateway amount
Paid To = ClickPay Clearing Account
Reference Doctype = Sales Invoice
Reference Name = Sales Invoice
Allocated Amount = confirmed gateway amount
```

The exact account field names must be adapted to the installed ERPNext
version.

Before submit:

-   invoice still exists;
-   invoice is submitted;
-   invoice is not cancelled;
-   amount does not exceed outstanding;
-   gateway payment has not already been accounted.

After submit:

-   reload invoice;
-   verify outstanding amount;
-   update local transaction to Paid.

## 15. Partial Payments

Each gateway payment is independent.

Example:

``` text
Sales Invoice: 100 KWD

Transaction A:
payment_id = P1
amount = 30
→ Payment Entry = 30
→ outstanding = 70

Transaction B:
payment_id = P2
amount = 20
→ Payment Entry = 20
→ outstanding = 50

Transaction C:
payment_id = P3
amount = 50
→ Payment Entry = 50
→ outstanding = 0
```

Never overwrite previous transactions.

## 16. Link Reuse

If an existing transaction is:

-   Link Created or Sent;
-   not expired;
-   still valid;
-   amount matches the requested amount;

then return the existing payment URL.

Do not create unnecessary duplicate links when the user simply clicks
Send Again.

## 17. New Link Generation

A new link is appropriate when:

-   previous link expired;
-   previous attempt failed/declined;
-   previous attempt was cancelled/voided;
-   a new partial amount is required;
-   user explicitly requests a new attempt.

Historical transactions remain unchanged.

## 18. Ten-Minute Expiry

Store:

``` text
created_at
expires_at = created_at + configured validity
```

Default:

``` text
10 minutes
```

The gateway's actual server-side expiration behavior must be confirmed
before production.

ERPNext must still prevent use/reuse of expired links according to
business rules.

## 19. Configuration

Create one `ClickPay Settings` singleton per site/company strategy as
appropriate.

Required configuration:

``` text
enabled
environment
base_url
client_id
client_secret
webhook_secret
webhook_url
success_url
error_url
link_validity_minutes
company
clearing_account
email_template
```

Secrets must use secure Frappe password/encrypted storage.

## 20. Logging

Log:

-   gateway endpoint category, not secrets;
-   HTTP status;
-   gateway request ID;
-   payment ID;
-   merchant reference;
-   transaction name;
-   error code;
-   safe error message;
-   processing duration.

Never log:

-   client secret;
-   webhook secret;
-   bearer token;
-   card data;
-   CVV;
-   authentication credentials.

## 21. Coding Standards

-   Use service classes/functions rather than putting all logic in
    DocType controllers.
-   Keep API calls isolated.
-   Use explicit exceptions.
-   Validate all external data.
-   Avoid circular imports.
-   Keep methods small and testable.
-   Use transactions carefully around Payment Entry creation.
-   Never use direct SQL updates to force invoice status.
-   Prefer Frappe ORM/document APIs for ERPNext accounting documents.