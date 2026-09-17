# KlickPay ERPNext Data Model

## 1. Standard ERPNext Documents

### Sales Invoice

Source of receivable.

Never directly set status to Paid.

### Payment Request

Represents customer payment request.

It does not replace Payment Entry accounting.

### Payment Entry

Actual accounting settlement.

### Customer

Source of customer identity/contact data.

### Company

Accounting company context.

### Account

Contains ClickPay Clearing Account.

## 2. ClickPay Transaction

Create a custom DocType named:

``` text
ClickPay Transaction
```

### Identity

  Field                Type                    Required
  -------------------- ----------------------- -------------
  name                 Autoname                Yes
  sales_invoice        Link: Sales Invoice     Yes
  payment_request      Link: Payment Request   Recommended
  customer             Link: Customer          Yes
  company              Link: Company           Yes
  merchant_reference   Data                    Yes

### Gateway Data

  Field                    Type              Required
  ------------------------ ----------------- ------------------------
  clickpay_payment_id      Data              After gateway creation
  clickpay_invoice_token   Data              Optional
  payment_url              Small Text/Data   After gateway creation
  clickpay_request_id      Data              Optional
  gateway_result           Data              Optional
  gateway_reference        Data              Optional
  gateway_auth_code        Data              Optional

### Amount

  Field                   Type
  ----------------------- --------------------------
  amount                  Currency
  currency                Data/Link as appropriate
  actual_payment_method   Data/Select
  requested_method        Select

`requested_method` should be:

``` text
GENERAL_LINK
```

### State

``` text
Created
Link Created
Sent
Pending
Paid
Failed
Declined
Voided
Expired
```

### Timing

``` text
created_at
expires_at
sent_at
paid_at
last_status_checked_at
```

### Accounting

``` text
payment_entry
```

Link to ERPNext Payment Entry.

### Diagnostics

``` text
error_code
error_message
retry_count
```

## 3. ClickPay Webhook Log

Create:

``` text
ClickPay Webhook Log
```

Fields:

``` text
event_type
payment_id
refund_reference
received_timestamp
gateway_timestamp
signature
signature_valid
idempotency_key
raw_payload
processing_status
processing_error
processed_at
```

Processing statuses:

``` text
Received
Queued
Processed
Failed
Rejected
Duplicate
```

## 4. ClickPay Settings

Create:

``` text
ClickPay Settings
```

Fields:

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

`client_secret` and `webhook_secret` must be secured/encrypted.

## 5. ClickPay Refund --- Phase 2

Fields:

``` text
sales_invoice
payment_entry
clickpay_payment_id
refund_reference
refund_type
refund_amount
refund_fee
original_fee_to_collect
total_amount_to_deduct
status
gateway_result
gateway_code
gateway_reference
gateway_payment_id
source
network
payment_channel
client_reference
event_type
created_at
processed_at
error_message
```

## 6. Indexing / Uniqueness

Strongly consider unique indexes/constraints for:

-   `clickpay_payment_id`;
-   `merchant_reference`;
-   webhook idempotency key.

The exact Frappe database constraint implementation should be selected
to fit the installed Frappe version.

## 7. Relationship

``` text
Sales Invoice
    │
    ├── Payment Request
    │       │
    │       └── ClickPay Transaction
    │                 │
    │                 ├── ClickPay Webhook Log(s)
    │                 │
    │                 └── Payment Entry
    │
    └── Payment Entry(s)
```

Multiple ClickPay Transactions and Payment Entries may belong to one
Sales Invoice.
