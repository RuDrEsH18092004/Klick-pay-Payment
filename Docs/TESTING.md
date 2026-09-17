# KlickPay ERPNext Integration --- Testing Specification

## 1. Testing Levels

1.  Unit tests
2.  Integration tests
3.  Webhook security tests
4.  ERPNext accounting tests
5.  End-to-end staging tests
6.  UAT
7.  Production smoke test

## 2. Unit Tests

### Payment Link

-   valid invoice;
-   invalid invoice;
-   cancelled invoice;
-   zero outstanding;
-   partial amount;
-   amount greater than outstanding;
-   invalid customer email;
-   missing phone;
-   unique reference;
-   reference \<= 50 characters;
-   response parsing;
-   payment URL preservation.

### Expiry

-   exactly before expiry;
-   exactly at expiry;
-   after expiry;
-   timezone handling.

### Idempotency

-   first success event;
-   duplicate event;
-   same payment with different event;
-   two concurrent workers.

## 3. Webhook Security Tests

### Valid

-   valid raw body;
-   valid secret;
-   valid timestamp;
-   valid signature.

Expected:

``` text
Accepted
Queued
Processed
```

### Invalid Signature

Change one payload byte.

Expected:

``` text
Rejected
No accounting
```

### Invalid Secret

Expected:

``` text
Rejected
```

### Replay

Use timestamp outside allowed window.

Expected:

``` text
Rejected
```

### JSON Formatting

Send equivalent JSON with different whitespace/order.

Signature must be based on the actual raw body received.

## 4. Accounting Tests

### Full Payment

``` text
Invoice = 100
Gateway = 100
```

Expected:

``` text
Payment Entry = 100
Outstanding = 0
Invoice = Paid
```

### Partial

``` text
Invoice = 100
Gateway = 50
```

Expected:

``` text
Payment Entry = 50
Outstanding = 50
Invoice = Partly Paid
```

### Multiple Partial

``` text
30 + 20 + 50
```

Expected:

``` text
3 Payment Entries
Outstanding = 0
```

### Duplicate

Same payment.success event twice.

Expected:

``` text
1 Payment Entry
```

## 5. Gateway Status Tests

Test:

``` text
paid + CAPTURED
failed
declined
voided
pending/unknown
```

Only confirmed successful payment may create accounting.

## 6. Amount Mismatch

Local:

``` text
50 KWD
```

Gateway:

``` text
60 KWD
```

Expected:

``` text
No Payment Entry
Exception logged
```

## 7. Currency Mismatch

Local:

``` text
KWD
```

Gateway:

``` text
unexpected currency
```

Expected:

``` text
No Payment Entry
Exception
```

## 8. Redirect Tests

### Redirect before webhook

Expected:

``` text
Payment verifying
```

No premature accounting.

### Webhook before redirect

Expected:

``` text
Payment accounted
```

Redirect displays verified result.

### Fake Success Redirect

Manually open success URL with fake success parameters.

Expected:

``` text
No Payment Entry
```

## 9. Link Tests

### Resend active link

Expected:

``` text
same payment_url
same payment_id
```

### Expired link

Expected:

``` text
Expired
new transaction when regenerated
```

### New failed attempt

Previous failed transaction must remain visible.

## 10. Error Recovery Tests

Test:

-   gateway timeout;
-   API 4xx;
-   API 5xx;
-   DNS/network failure;
-   status API timeout;
-   webhook queue failure;
-   Payment Entry validation failure;
-   database concurrency.

## 11. Security Tests

Verify:

-   no credentials in browser;
-   no access token in frontend;
-   no secrets in logs;
-   authorized users only;
-   HTTPS;
-   webhook HMAC;
-   replay protection;
-   safe error messages.

## 12. UAT Acceptance Matrix

  ID       Scenario                              Expected
  -------- ------------------------------------- ---------------------
  UAT-01   100 KWD full payment                  Invoice Paid
  UAT-02   50 KWD partial payment                Invoice Partly Paid
  UAT-03   Second partial payment                Invoice Paid
  UAT-04   Declined                              No Payment Entry
  UAT-05   Failed                                No Payment Entry
  UAT-06   Duplicate webhook                     One Payment Entry
  UAT-07   Invalid signature                     Rejected
  UAT-08   Stale timestamp                       Rejected
  UAT-09   Redirect only                         No accounting
  UAT-10   Active resend                         Same link reused
  UAT-11   Expired link                          New link generated
  UAT-12   Amount mismatch                       Accounting blocked
  UAT-13   Gateway confirmed / ERPNext failure   Retryable exception
  UAT-14   Clearing account                      Correct ledger
  UAT-15   Multiple attempts                     Full audit trail

## 13. Production Smoke Test

After production deployment:

1.  Create a controlled low-value Sales Invoice.
2.  Generate payment link.
3.  Verify email.
4.  Open checkout.
5.  Complete payment.
6.  Confirm webhook.
7.  Confirm status.
8.  Confirm Payment Entry.
9.  Confirm clearing account.
10. Confirm invoice status.
11. Record payment_id and settlement/reconciliation evidence.
