# KlickPay ERPNext Integration --- Deployment Guide

## 1. Prerequisites

-   ERPNext/Frappe environment;
-   custom app deployment access;
-   staging KlickPay credentials;
-   production KlickPay credentials;
-   confirmed API URLs;
-   webhook secret;
-   public HTTPS domain;
-   success URL;
-   error URL;
-   ClickPay Clearing Account;
-   email configuration.

## 2. Staging Setup

Configure:

``` text
Environment = Staging
Base URL = https://api.staging.klick-pay.com/v3
Enabled = Yes
```

Set:

``` text
client_id
client_secret
webhook_secret
webhook_url
success_url
error_url
link_validity_minutes = 10
company
clearing_account
email_template
```

## 3. Install App

Recommended sequence:

``` bash
bench get-app <repository-or-source>
bench --site <site> install-app clickpay_integration
bench --site <site> migrate
bench --site <site> clear-cache
bench restart
```

Use the project's actual repository/app source and standard deployment
process.

## 4. Permissions

Create/review permissions for:

-   ClickPay Settings;
-   ClickPay Transaction;
-   ClickPay Webhook Log;
-   ClickPay Refund.

Do not allow ordinary users to view encrypted credentials unnecessarily.

## 5. Public URLs

Success:

``` text
https://<erp-domain>/clickpay/success
```

Error:

``` text
https://<erp-domain>/clickpay/error
```

Webhook:

``` text
https://<erp-domain>/<approved-webhook-route>
```

All must use HTTPS.

## 6. Webhook Configuration

Configure KlickPay to send required events:

``` text
payment.success
payment.declined
payment.failed
payment.voided
```

Phase 2:

``` text
refund.success
refund.failed
refund.pending_balance
refund.under_review
```

## 7. Pre-Production Checklist

-   [ ] Production API URL confirmed.
-   [ ] Production credentials received.
-   [ ] Webhook secret confirmed.
-   [ ] Webhook retry behavior confirmed.
-   [ ] Timestamp tolerance confirmed.
-   [ ] 10-minute expiry behavior confirmed.
-   [ ] General Link behavior confirmed.
-   [ ] Partial-payment behavior confirmed.
-   [ ] ClickPay Clearing Account created.
-   [ ] Email template approved.
-   [ ] UAT passed.
-   [ ] Backup completed.
-   [ ] Rollback procedure tested.

## 8. Production Cutover

1.  Backup ERPNext.
2.  Deploy application.
3.  Run migrations.
4.  Configure production settings.
5.  Verify HTTPS.
6.  Verify webhook endpoint.
7.  Configure KlickPay webhook.
8.  Enable integration.
9.  Create controlled invoice.
10. Generate low-value link.
11. Complete controlled payment.
12. Verify webhook.
13. Verify payment status.
14. Verify Payment Entry.
15. Verify clearing account.
16. Verify invoice status.
17. Monitor logs.

## 9. Rollback

Disable new payment-link generation.

Do not delete:

-   ClickPay Transactions;
-   Webhook Logs;
-   Payment Entries;
-   gateway identifiers.

Existing confirmed gateway payments must still be reconciled.

If automated accounting is disabled, maintain an exception list of
confirmed gateway payments awaiting ERPNext accounting.

## 10. Monitoring

Monitor:

``` text
payment link creation failures
webhook rejection rate
webhook processing failures
pending transactions
paid-but-not-accounted transactions
Payment Entry failures
amount mismatches
currency mismatches
expired links
```

## 11. Production Support

For every incident collect:

``` text
Sales Invoice
ClickPay Transaction
merchant_reference
payment_id
request_id
event_type
gateway status
gateway result
Payment Entry
timestamps
```

Never request/store customer card credentials.
