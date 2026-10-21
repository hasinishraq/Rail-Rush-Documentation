# Payments

## Start a payment

`POST /v1/payments` · user + `Idempotency-Key` · `201 Created`

```json
{ "pnr": "9072567897", "method": "BKASH", "return_url": "http://localhost:8088/pay/return" }
```

| Field | Rules |
|---|---|
| `pnr` | 10 digits, your own hold, OTP verified |
| `method` | `BKASH`, `NAGAD`, `ROCKET`, `UPAY`, `VISA`, `MASTERCARD`, `NEXUS` |
| `return_url` | Must start with an allow-listed origin (`RETURN_URL_ALLOWLIST`) |

```json
{
  "payment_id": "6a0c6f7e-8d55-4f53-9d0b-2f5c1a7d9e10",
  "pnr": "9072567897",
  "status": "INITIATED",
  "method": "BKASH",
  "amount": { "amount_minor": 40000, "currency": "BDT" },
  "redirect_url": "http://localhost:8090/pay/ord_43ce4a0c-56fc",
  "hold_expires_at": "2026-10-20T09:40:00Z"
}
```

Send the user to `redirect_url`. It works once. After paying, the gateway returns the user to
`return_url`; the app then polls the booking until it is `CONFIRMED`.

| Error | When |
|---|---|
| `409 OTP_REQUIRED` | The OTP has not been verified |
| `409 HOLD_EXPIRED` | The hold ran out |
| `409 ALREADY_PAID` | The PNR is already paid |
| `403 FORBIDDEN` / `404 NOT_FOUND` | Not your PNR / unknown PNR |
| `422 VALIDATION_FAILED` | Bad method or `return_url` |

Only one live payment (`INITIATED` or `CAPTURED`) can exist per PNR.

## Payment status

`GET /v1/payments/{paymentId}` · owner or staff → the payment, including `captured_at` and
`booking_status` once known.

## Gateway webhook

`POST /v1/webhooks/payment/{gateway}` · called by the gateway, not by clients

- The body must carry a valid signature: for the fake gateway, `X-Signature` = hex
  HMAC-SHA256 of the raw body with `GATEWAY_SECRET`. A bad signature gives `401` and increments
  `webhook_signature_failures_total`.
- Each delivery is applied once (`webhook_event` primary key); repeats return `200 {"ok": true}`.
- A capture whose amount or currency differs from the payment is marked `FAILED`, alerted, and refunded.
- A capture for a PNR that already has another live payment is refunded (`DUPLICATE_CAPTURE`).
- A valid capture writes a `Confirm` command to the outbox in the same transaction.

Unknown gateway names give `404`. The gateway retries on any `5xx`.

## Adding a real gateway

Implement `payment.GatewayAdapter` (`internal/payment/gateway.go`): create order, verify webhook
signature (HMAC or RSA per provider), query status (for the reconciler), refund. Register it in
`GATEWAYS` and configure its base URL and secret.

## Local fake gateway

| Request | Effect |
|---|---|
| `GET :8090/pay/{order_id}` | The pay page the user is redirected to |
| `POST :8090/orders/{order_id}/pay?outcome=success\|fail\|unknown` | Settle from a script; `unknown` sends no webhook, so the reconciler must find it |

The fake gateway also injects random failures and unknown statuses (`FAIL_RATE`, `UNKNOWN_RATE`,
5 % each locally) and delays webhooks (`DELAY`, 500 ms).
