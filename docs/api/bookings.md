# Bookings and tickets

## My bookings

`GET /v1/me/bookings` · user

```json
{
  "items": [{
    "pnr": "6910567903", "status": "CONFIRMED", "train_no": "742", "run_date": "2026-10-21",
    "from": { "idx": 1, "station_code": "BAN", "station_name": "Dhaka Biman Bandar", "time": "2026-10-21T17:43:00Z" },
    "to":   { "idx": 4, "station_code": "CML", "station_name": "Cumilla", "time": "2026-10-21T21:13:00Z" },
    "seats": [{ "seat_id": "CHA-37", "seat_idx": 220, "coach": "CHA", "seat_no": 37 }]
  }],
  "next_cursor": "…"
}
```

Newest first; pass `next_cursor` back to get the next page.

## A booking

`GET /v1/bookings/{pnr}` · owner, or staff (`counter`, `agent`, `admin`)

```json
{
  "pnr": "6910567903",
  "status": "CONFIRMED",
  "journey_id": 74220261021,
  "train_no": "742", "train_name": "Turna", "run_date": "2026-10-21",
  "from": { "...": "StopRef" }, "to": { "...": "StopRef" },
  "class_code": "S_CHAIR",
  "passengers": [{ "seq": 1, "name": "Test Pax", "gender": "MALE", "type": "ADULT", "status": "ISSUED",
                   "seat": { "seat_id": "CHA-37", "seat_idx": 220, "coach": "CHA", "seat_no": 37 },
                   "fare": { "amount_minor": 24000, "currency": "BDT" } }],
  "amount": { "amount_minor": 26000, "currency": "BDT" },
  "payment": { "payment_id": "…", "status": "CAPTURED", "method": "BKASH", "amount": { "amount_minor": 26000, "currency": "BDT" } },
  "refunds": [],
  "updated_at": "2026-10-20T12:51:30Z"
}
```

Somebody else's PNR gives `404`, never `403`, so PNRs cannot be probed.

### Booking status

| Status | Meaning |
|---|---|
| `PENDING_OTP` | Hold created; waiting for the OTP |
| `PENDING_PAYMENT` | OTP verified; waiting for payment (`hold_expires_at`) |
| `CONFIRMED` | Paid; tickets issued |
| `PARTIALLY_CANCELLED` | Some passengers cancelled |
| `CANCELLED` | All passengers cancelled |
| `EXPIRED` | Hold ran out unpaid |
| `CONFIRM_FAILED` | Paid too late with no seats left; fully refunded |

`seats_changed: true` means the payment arrived just after the hold ended and the passenger got the
nearest free seats instead of the original ones.

### Ticket (passenger) status

`HELD`, `ISSUED`, `CANCELLED`, `EXPIRED`, `FAILED`.

## Cancel

`POST /v1/bookings/{pnr}/cancel` · owner (or `counter` staff) + `Idempotency-Key`

```json
{ "passenger_seqs": [2], "reason": "plans changed" }
```

Leave `passenger_seqs` empty to cancel the whole PNR. Response (`200`, or `202` when the allocator is slow):

```json
{
  "request_id": "…",
  "pnr": "6910567903",
  "cancel_id": "<the Idempotency-Key>",
  "cancelled_seqs": [2],
  "booking_status": "PARTIALLY_CANCELLED",
  "passengers": [{ "...": "PassengerSeat" }],
  "refund_estimate": { "amount_minor": 18000, "currency": "BDT" },
  "refund_status": "PENDING"
}
```

The refund is paid asynchronously by the refunder, following the [refund rules](../business-rules.md#refunds).

| Error | When |
|---|---|
| `409 NOT_CANCELLABLE` | Status is not `CONFIRMED` / `PARTIALLY_CANCELLED`, or the train has departed (`details.reason`) |
| `422 VALIDATION_FAILED` | Unknown or repeated passenger seq |
| `404 NOT_FOUND` | Unknown PNR, or not yours |

## E-ticket PDF

`GET /v1/bookings/{pnr}/ticket` · owner or staff · `application/pdf`

Only for `CONFIRMED` and `PARTIALLY_CANCELLED` bookings. A one-page A4 ticket with the journey card,
a signed QR code, trip details, passengers, fare breakdown and travel notes. Names in Bangla are
shaped with HarfBuzz (`go-text/typesetting`) and drawn in Hind Siliguri. Cached privately for 60 s.

## Ticket QR code

The QR code holds the issued seats of the booking, signed with ES256 (`TICKET_SIGNING_KEY_FILE`):

```text
RAILBD2|<pnr>|<train_no>|<run_date>|<FROM>-<TO>|<seat,seat,...>|<kid>|<signature>
```

The signature is base64url `r‖s` over everything before the last `|`. Changing any field
invalidates it.

### Public keys

`GET /v1/tickets/keys` · public · JWKS

Gate scanners verify signatures offline with these keys, selected by `kid`.

### Verify a ticket

`POST /v1/tickets/verify` · staff (`counter`, `agent`, `admin`)

```json
{ "qr": "RAILBD2|6910567903|742|2026-10-21|BAN-CML|CHA-37|t1|MEUCIQ..." }
```

Checks the signature, then the booking's current state, so a seat cancelled after printing is
reported as cancelled.

```json
{
  "valid": true,
  "pnr": "6910567903",
  "booking_status": "CONFIRMED",
  "train_no": "742", "train_name": "Turna", "run_date": "2026-10-21",
  "from": { "...": "StopRef" }, "to": { "...": "StopRef" },
  "class_code": "S_CHAIR",
  "passengers": [{ "seq": 1, "name": "Test Pax", "status": "ISSUED", "seat": { "seat_id": "CHA-37", "...": "SeatRef" } }]
}
```

`valid` is `true` when at least one seat on the ticket is still `ISSUED`.

| Error | When |
|---|---|
| `422 TICKET_INVALID` | Malformed payload, unknown key, bad signature, unknown PNR or trip mismatch (`details.reason`) |
| `403 FORBIDDEN` | Caller is not staff |
