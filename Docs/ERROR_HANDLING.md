# KlickPay Error Handling and Recovery

## 1. Error Classification

### Validation Errors

Examples:

-   invoice not submitted;
-   invoice cancelled;
-   zero outstanding;
-   invalid amount;
-   missing customer data.

Action:

``` text
Reject before gateway call.
```

### Gateway Errors

Examples:

-   invalid credentials;
-   4xx;
-   5xx;
-   invalid request.

Action:

``` text
Log safe diagnostic data.
Keep invoice outstanding.
```

### Network Errors

Special handling is required for payment creation.

A timeout does not necessarily mean KlickPay did not create the payment.

Never automatically issue repeated payment creation requests without
reconciliation.

### Webhook Errors

Invalid signature:

``` text
Reject.
```

Stale timestamp:

``` text
Reject.
```

Unknown payment:

``` text
Persist and investigate.
```

Processing exception:

``` text
Persist.
Queue retry.
```

## 2. Gateway Paid but ERPNext Failed

State:

``` text
Gateway = Paid
ERPNext = Not Accounted
```

This is an operational exception.

Retry accounting safely.

Do not create another gateway payment.

## 3. Duplicate Webhook

If already processed:

``` text
No new Payment Entry.
```

The second event should be recorded as Duplicate.

## 4. Amount Mismatch

Never silently adjust.

Example:

``` text
Local requested = 50
Gateway confirmed = 60
```

Action:

``` text
Block accounting
Mark exception
Alert finance/support
```

## 5. Invoice Already Paid

If a late success webhook arrives after another valid Payment Entry has
already closed the invoice:

-   do not create a duplicate Payment Entry;
-   preserve the gateway transaction;
-   flag it for investigation if the external payment represents an
    additional real receipt.

## 6. Invoice Cancelled

If payment occurs against an invoice that was cancelled after link
creation:

-   preserve transaction;
-   do not blindly post accounting;
-   raise finance exception;
-   follow approved refund/reallocation process.

## 7. Retry Strategy

Retry automatically when the operation is known to be safe.

Use exponential backoff for transient background failures.

Do not retry forever.

After configurable attempts:

``` text
Failed / Manual Review
```

## 8. Customer Messages

Never show raw API errors.

Use:

``` text
Payment could not be created. Please try again.
```

or:

``` text
Payment is being verified. Please check again shortly.
```

or:

``` text
Payment could not be confirmed. Please contact support with your invoice number.
```
