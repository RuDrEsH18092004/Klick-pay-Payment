# KlickPay ERPNext Accounting Specification

## 1. Accounting Principle

KlickPay confirms the external payment.

ERPNext Payment Entry confirms the receivable settlement.

Therefore:

``` text
KlickPay success
      ↓
Server-side verification
      ↓
ERPNext Payment Entry
      ↓
Sales Invoice outstanding recalculated
```

## 2. Never Directly Set Invoice Paid

Do not implement:

``` python
invoice.status = "Paid"
```

as the accounting mechanism.

ERPNext must calculate invoice status from submitted accounting
documents.

## 3. Full Payment

Invoice:

``` text
100 KWD
```

Gateway confirms:

``` text
100 KWD
```

Payment Entry:

``` text
Payment Type: Receive
Amount: 100 KWD
Party Type: Customer
Party: Invoice Customer
Paid To: ClickPay Clearing Account
Reference: Sales Invoice
Allocated Amount: 100 KWD
```

After submission:

``` text
Outstanding = 0 KWD
Status = Paid
```

## 4. Partial Payment

Invoice:

``` text
100 KWD
```

Gateway confirms:

``` text
50 KWD
```

Payment Entry:

``` text
50 KWD
```

After submission:

``` text
Outstanding = 50 KWD
Status = Partly Paid
```

## 5. Multiple Payments

Example:

``` text
Payment 1 = 30
Payment 2 = 20
Payment 3 = 50
```

ERPNext:

``` text
100 invoice
-30
-20
-50
=0 outstanding
```

Three independent Payment Entries are expected.

## 6. Clearing Account

Use:

``` text
ClickPay Clearing Account
```

as the receiving/paid-to account.

This separates:

``` text
Customer payment confirmation
```

from:

``` text
Actual gateway settlement to bank
```

## 7. Settlement

Settlement is a separate finance process.

Typical conceptual flow:

``` text
Customer
   ↓ 100
KlickPay
   ↓
ClickPay Clearing Account in ERPNext
   ↓ settlement/reconciliation
Bank
```

The integration should not assume that customer payment date equals bank
settlement date.

## 8. Fees

Gateway fees are merchant-side.

Do not inflate the customer's invoice payment amount merely to account
for merchant fees.

If fee accounting is required:

``` text
Customer payment = invoice allocation
Gateway fee = separate expense/fee accounting
```

Exact fee accounting must be approved by Finance and based on KlickPay
settlement data.

## 9. Payment Entry Safety Checks

Before creation:

-   invoice is submitted;
-   invoice is not cancelled;
-   outstanding amount \> 0;
-   confirmed amount \> 0;
-   confirmed amount \<= outstanding;
-   currency matches;
-   payment_id exists;
-   transaction exists;
-   transaction is not already accounted;
-   no Payment Entry already exists for the gateway payment.

## 10. Payment Entry Failure

If KlickPay says paid but Payment Entry creation/submission fails:

Do not mark the gateway transaction as fully accounted.

Instead:

``` text
Gateway = Confirmed Paid
ERPNext Accounting = Exception
```

Queue a retry and alert the support/finance team.

Do not ask a user to manually mark the invoice paid.

## 11. Cancellation Race Condition

If an invoice is cancelled after payment link creation but before
payment:

-   payment link remains an external object;
-   webhook may still arrive;
-   system must not blindly create accounting against a cancelled
    invoice;
-   create an operational exception for finance/support;
-   preserve the external payment audit trail.

The exact business response for this edge case must be approved by
Finance.
