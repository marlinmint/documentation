# Marlin Mint API

This document describes the Marlin Mint API: available endpoints, request and response formats, status values, error codes, and webhook signature verification. It is intended for developers integrating payment intake and payout flows.

## Base URL
```
https://api.marlinmint.com/v1
```

Integration (test) environment:
```
https://api-test.marlinmint.com/v1
```

## Authentication
All requests require an `Authorization` header with your API key.

```
Authorization: <your-api-key>
Content-Type: application/json
```

---

## Create Payment (Intake)

**Endpoint:** `POST /payments/checkout`

### Request (redirect to payment page)
```json
{
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "flow_type": "intake",
    "rail": "p2p",
    "order_ref": "order_12345",
    "payer_ref": "user_001",
    "payer_name": "Іван Петренко",
    "hook_url": "https://your-site.com/webhook",
    "done_url": "https://your-site.com/success"
}
```

### Request with credentials (no_ui)
Server-to-server, without our payment page: the response will contain the
transfer details, which you display to the payer yourself.

```json
{
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "flow_type": "intake",
    "rail": "p2p",
    "order_ref": "order_1233272245",
    "payer_ref": "user_001",
    "payer_name": "Іван Петренко",
    "hook_url": "https://your-site.com/webhook",
    "no_ui": true
}
```

### Request with card (rail: acquiring)
```json
{
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "flow_type": "intake",
    "rail": "acquiring",
    "order_ref": "order_123452292027",
    "payer_ref": "user_001",
    "hook_url": "https://your-site.com/webhook",
    "no_ui": true,
    "intake_details": {
        "issuer": "Private Bank",
        "card_input": {
            "card_number": "5246081344140382",
            "expiry_month": 2,
            "expiry_year": 29,
            "security_code": "908"
        }
    }
}
```

### Response
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "open",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": "https://pay.marlinmint.com/pay_in_abc123",
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": null,
    "sum_settled": null,
    "failure_note": null,
    "valid_till": null,
    "bank_details": null
}
```

---

## Create Payment (Payout)

**Endpoint:** `POST /payments/checkout`

### Payout to card
```json
{
    "sum_total": "500.00",
    "currency_iso": "UAH",
    "flow_type": "payout",
    "rail": "p2p",
    "order_ref": "withdraw_67890",
    "payer_ref": "user_001",
    "hook_url": "https://your-site.com/webhook",
    "remit_details": {
        "to_name": "Іван Петренко",
        "dest_account": "5246081344140382"
    }
}
```

### Response
```json
{
    "txn_ref": "pay_out_xyz789",
    "order_ref": "withdraw_67890",
    "stage": "open",
    "sum_total": "500.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": null,
    "started_at": "2026-07-31T11:00:00.000Z",
    "finished_at": null,
    "sum_settled": null,
    "failure_note": null,
    "valid_till": null,
    "bank_details": null
}
```

---

## Check Payment Status

**Endpoint:** `GET /payments/{txn_ref}/lookup`

```
GET /payments/pay_in_abc123/lookup
```

### Response — processing, credentials issued
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "authorized",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": "https://pay.marlinmint.com/pay_in_abc123",
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": null,
    "sum_settled": null,
    "failure_note": null,
    "bank_details": {
        "type": "card",
        "card_number": "4149629311879546",
        "beneficiary_name": "Приват*Іван П."
    }
}
```

### Response — success
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "cleared",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": "https://pay.marlinmint.com/pay_in_abc123",
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": "2026-07-31T10:35:00.000Z",
    "sum_settled": "970.00",
    "failure_note": null,
    "bank_details": null
}
```

### Response — declined
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "unsettled",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "pay_page_url": null,
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": null,
    "sum_settled": null,
    "failure_note": "Payment timeout",
    "bank_details": null
}
```

---

## Check Balance

**Endpoint:** `GET /partner/holdings`

```json
{
    "amount_available": "15000.00",
    "ccy_code": "UAH"
}
```

---

## Raise Claim

**Endpoint:** `POST /payments/{txn_ref}/claim`

Disputes the result of a payment. Body — `multipart/form-data`: 1 to 3
attachments (up to 50 MB each) and an optional `comment`. Response:

```json
{
    "outcome": "accepted"
}
```

---

## Stage Values

The same five states shown in the merchant portal and the CSV export.

| Stage | Description |
|-------|--------------|
| `open` | created, payer has not yet confirmed the transfer |
| `authorized` | funds confirmed, settlement in progress |
| `cleared` | completed in full |
| `part_cleared` | completed partially, less than requested was received |
| `unsettled` | did not go through: decline, cancellation, expiry, reversal, or chargeback |

Terminal states — `cleared`, `part_cleared`, `unsettled`. Any other state can
still change; poll the status or wait for the webhook until a terminal state
is reached.

---

## Error Response Format

```json
{
    "outcome": "rejected",
    "problem": {
        "reason": "invalid_input",
        "detail": "sum_total is required",
        "invalid_fields": null
    }
}
```

| Reason | Description | HTTP |
|--------|--------------|------|
| `invalid_input` | invalid request parameters | 400, 422 |
| `payment_rejected` | payment processing error | 400 |
| `not_authorized` | authentication error or domain not allowed | 401, 403 |
| `internal_failure` | internal error | 500 |

`invalid_fields` is populated only for request body validation errors — a
list of fields with a description of the issue, otherwise `null`.

---

## Webhook Notification

A POST request is sent to your `hook_url` whenever the stage changes.

### Signature Verification (RSA SHA-256)

Each callback can be verified using an RSA SHA-256 signature to confirm
authenticity and integrity.

**Verification flow:**

1. Extract the raw request body exactly as received (no modifications).
2. Calculate the SHA-256 hash of the body.
3. Verify the signature using the public key.

**Signature header:**
```
X-Signature
```

**Verification rules:**

* Algorithm: RSA + SHA256
* Encoding: Base64
* Any callback with an invalid signature must be rejected

### Example payload
```json
{
    "txn_ref": "pay_in_abc123",
    "order_ref": "order_12345",
    "stage": "cleared",
    "sum_total": "1000.00",
    "currency_iso": "UAH",
    "network": null,
    "started_at": "2026-07-31T10:30:00.000Z",
    "finished_at": "2026-07-31T10:35:00.000Z",
    "sum_settled": "970.00",
    "failure_note": null
}
```

Respond with `200` — otherwise delivery will be retried. Your handler must be
idempotent by `txn_ref`: a single payment may generate several notifications
(e.g. `authorized`, then `cleared`).

---

## Field Reference

### Request Fields

| Field | Type | Required | Description |
|-------|------|----------|--------------|
| `sum_total` | string | yes | transaction amount |
| `currency_iso` | string | yes | ISO 4217 code (`UAH`) |
| `flow_type` | string | yes | `intake` (deposit) or `payout` (withdrawal) |
| `rail` | string | yes | `p2p` (card-to-card / bank transfer) or `acquiring` (card acquiring) |
| `network` | string | no | payment network, if applicable |
| `order_ref` | string | yes | your unique reference, 5–64 characters |
| `payer_ref` | string | yes | payer identifier on your side |
| `payer_name` | string | no | payer full name |
| `hook_url` | string | yes | webhook URL |
| `done_url` | string | conditional | required if `no_ui` is not set |
| `no_ui` | boolean | no | mode without our payment page |
| `intake_details` | object | no | deposit credentials |
| `remit_details` | object | no | withdrawal credentials |

`order_ref` is the idempotency key. A repeated request with the same
`order_ref` returns the existing payment instead of creating a new one.

### intake_details (deposit)

| Field | Type | Description |
|-------|------|--------------|
| `issuer` | string | bank name for P2P matching |
| `card_input` | object | card details for `rail: acquiring` |

### card_input

| Field | Type | Description |
|-------|------|--------------|
| `card_number` | string | card number |
| `expiry_month` | number | expiry month |
| `expiry_year` | number | expiry year (2 digits) |
| `security_code` | string | CVV/CVC |

### remit_details (withdrawal)

| Field | Type | Description |
|-------|------|--------------|
| `dest_account` | string | card number, 16 digits |
| `to_name` | string | recipient full name |
| `iban_code` | string | IBAN for international transfers |
| `tax_number` | string | tax ID |

### Response Fields

| Field | Type | Description |
|-------|------|--------------|
| `txn_ref` | string | our payment identifier |
| `order_ref` | string | your identifier from the request |
| `stage` | string | payment stage, see Stage Values |
| `sum_total` | string | requested amount |
| `sum_settled` | string \| null | amount actually settled |
| `currency_iso` | string | ISO 4217 code |
| `network` | string \| null | payment network |
| `pay_page_url` | string \| null | link to the payment page |
| `started_at` | string | creation time, ISO 8601 |
| `finished_at` | string \| null | completion time, ISO 8601 |
| `failure_note` | string \| null | decline reason |
| `valid_till` | string \| null | payment expiry, if set |
| `bank_details` | object \| null | transfer details in `no_ui` mode |
