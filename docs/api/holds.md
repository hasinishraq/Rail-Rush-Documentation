# Holds and OTP

## Create a hold

`POST /v1/holds` · user + admission + `Idempotency-Key` · `201 Created` (or `202`, see
[asynchronous results](selection.md#asynchronous-results))

```json
{
  "journey_id": 74220261021,
  "passengers": [
    { "name": "Ayesha Rahman", "gender": "FEMALE", "type": "ADULT" },
    { "name": "আয়ান রহমান", "gender": "MALE", "type": "CHILD" }
  ]
}
```

One passenger per locked seat, **in cart order**. The locked seats become a hold with a PNR; an OTP is
texted to the account's phone.

```json
{
  "request_id": "01a15955-d722-79a8-b492-a8be72462363",
  "pnr": "9072567897",
  "status": "PENDING_OTP",
  "journey_id": 74220261021,
  "train_name": "Turna (742)",
  "from": { "idx": 1, "station_code": "BAN", "station_name": "Dhaka Biman Bandar", "time": "2026-10-21T17:43:00Z" },
  "to":   { "idx": 4, "station_code": "CML", "station_name": "Cumilla", "time": "2026-10-21T21:13:00Z" },
  "class_code": "S_CHAIR",
  "passengers": [{ "seq": 1, "name": "Ayesha Rahman", "gender": "FEMALE", "type": "ADULT", "status": "HELD",
                   "seat": { "seat_id": "CHA-10", "seat_idx": 193, "coach": "CHA", "seat_no": 10 },
                   "fare": { "amount_minor": 24000, "currency": "BDT" } }],
  "fare":           { "amount_minor": 36000, "currency": "BDT" },
  "service_charge": { "amount_minor": 4000, "currency": "BDT" },
  "amount":         { "amount_minor": 40000, "currency": "BDT" },
  "expires_at": "2026-10-20T09:40:00Z",
  "otp": { "sent_to": "+88017•••••111", "expires_at": "2026-10-20T09:33:00Z", "resend_after": "2026-10-20T09:31:00Z" }
}
```

Children pay half fare. The hold lasts 10 minutes (`expires_at`); OTP and payment must both happen
before it ends. Retrying with the same `Idempotency-Key` returns the same PNR.

| Error | When |
|---|---|
| `409 EMPTY_CART` | The locks expired before the details were sent |
| `422 PASSENGER_COUNT_MISMATCH` | Not exactly one passenger per locked seat |
| `409 ACTIVE_HOLD_EXISTS` | You already hold seats on this train (`details.pnr`) |
| `403 ADMISSION_USED` | This admission already produced a booking (`details.pnr`) |
| `422 VALIDATION_FAILED` | Bad name, gender or type (`details.fields`) |

## Verify the OTP

`POST /v1/holds/{pnr}/otp/verify` · user

```json
{ "otp": "482913" }
```

```json
{ "pnr": "9072567897", "status": "PENDING_PAYMENT", "expires_at": "2026-10-20T09:40:00Z" }
```

| Error | When |
|---|---|
| `422 OTP_INVALID` | Wrong code; `details.attempts_left` |
| `409 OTP_EXPIRED` | The 3-minute code expired; ask for a new one |
| `429 OTP_LOCKED` | 5 wrong codes; ask for a new one |
| `409 HOLD_EXPIRED` | The hold itself ran out |
| `404 NOT_FOUND` | Unknown PNR, or not yours |

Verifying an already verified PNR succeeds again (idempotent).

!!! warning "Local development"
    When `OTP_DEV_CODE` is set (empty in `deploy/.env.example`; you choose it), that code is also accepted. Never
    set it in production.

## Resend the OTP

`POST /v1/holds/{pnr}/otp/resend` · user → the new `otp` info.

A new code replaces the old one and resets the attempts. Limits: 60 s after the last send, at most 3
resends per PNR, at most 10 sends per user per hour (`429 OTP_LOCKED` with `details.retry_after`).
