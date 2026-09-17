# KlickPay Refund Architecture --- Phase 2

## 1. Purpose

Support full and partial refunds of confirmed KlickPay payments while
keeping ERPNext accounting and KlickPay gateway state synchronized.

## 2. Endpoints

Full refund:

``` http
POST /refunds/full
```

Partial refund:

``` http
POST /refunds/partial
```

## 3. Full Refund

Input:

``` text
payment_id
```

The refund must reference the original successful payment.

## 4. Partial Refund

Input:

``` text
payment_id
refund_amount
```

Validate:

``` text
refund_amount > 0
refund_amount <= refundable amount
```

## 5. Refund Lifecycle

``` text
Refund Requested
    ↓
KlickPay API
    ↓
Successful
    OR
Pending Collection
    OR
Under Review
    OR
Failed
    ↓
Webhook confirmation
    ↓
ERPNext refund accounting
```

## 6. Pending Balance

KlickPay may return an insufficient-balance condition.

The refund may become:

``` text
pending_collection
```

KlickPay can automatically retry after merchant funding according to its
process.

ERPNext should not repeatedly submit the same refund request blindly.

## 7. Refund Data

Store:

``` text
original payment_id
refund_reference
refund_type
refund_amount
refund_fee
original_fee_to_collect
total_amount_to_deduct
refund_status
gateway_result
gateway_code
gateway_reference
gateway_payment_id
```

## 8. Webhooks

Support:

``` text
refund.success
refund.failed
refund.pending_balance
refund.under_review
```

## 9. ERPNext Accounting

Refund accounting must be designed around ERPNext's supported Sales
Invoice return/refund mechanism and the original Payment Entry.

The exact implementation should be approved by Finance before
development because the correct ERPNext accounting treatment depends on
whether the business is cancelling/returning goods, reversing
receivable, or refunding an overpayment.

## 10. Important Rule

A successful gateway refund does not by itself determine the correct
ERPNext accounting document.

Both gateway state and ERPNext accounting state must be tracked.
