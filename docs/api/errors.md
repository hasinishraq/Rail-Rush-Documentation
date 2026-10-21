# Error codes

All errors use the [error envelope](index.md#errors). Codes are stable; messages may change.

## General

| Code | HTTP | Meaning |
|---|---|---|
| `BAD_REQUEST` | 400 | Malformed JSON, missing or non-UUID `Idempotency-Key` |
| `VALIDATION_FAILED` | 422 | Field errors in `details.fields` |
| `NOT_FOUND` | 404 | Unknown resource, or one you may not see |
| `FORBIDDEN` | 403 | Authenticated but not allowed |
| `IDEMPOTENCY_KEY_REUSED` | 422 | Same key, different request body |
| `REQUEST_IN_PROGRESS` | 409 | Same key, first request still running |
| `RATE_LIMITED` | 429 | Too many requests; `details.retry_after` |
| `OVERLOADED` | 429 | Pod or partition is saturated; retry shortly |
| `LOG_UNAVAILABLE` | 503 | Kafka unavailable; **nothing was changed**, safe to retry |
| `UNAVAILABLE` | 503 | A dependency is down |
| `INTERNAL` | 500 | Unexpected error; quote `request_id` |

## Authentication

| Code | HTTP | Meaning |
|---|---|---|
| `UNAUTHENTICATED` | 401 | Missing or invalid token, or bad webhook signature |
| `TOKEN_EXPIRED` | 401 | Access token expired; refresh it |
| `INVALID_CREDENTIALS` | 401 | Wrong login or password (also for unknown or locked users) |
| `REFRESH_REUSED` | 401 | A used refresh token was replayed; the session family is revoked |
| `CAPTCHA_INVALID` | 403 | Turnstile token missing, expired, already used or rejected; get a new one and retry |

## Waiting room

| Code | HTTP | Meaning |
|---|---|---|
| `ADMISSION_REQUIRED` | 403 | No `X-Admission-Token`; `details.queue_url` |
| `ADMISSION_INVALID` | 403 | Expired, badly signed, or bound to another user |
| `ADMISSION_USED` | 403 | Already used for a booking; `details.pnr` |

## Seat selection

| Code | HTTP | Meaning |
|---|---|---|
| `NOT_OPEN` | 409 | Sale window not open; `details.opens_at` |
| `SALES_CLOSED` | 409 | Online sales closed |
| `SOLD_OUT` | 409 | No seat left in the class for the trip |
| `SEAT_TAKEN` | 409 | Seat taken on part of the trip; `details.seat_id` |
| `SEAT_NOT_IN_POOL` | 422 | Seat belongs to another class or quota |
| `SELECTION_LIMIT` | 409 | Over the per-journey seat limit; `details.max` |
| `ORDER_LIMIT_EXCEEDED` | 409 | Over the per-day seat limit; `details.limit`, `details.issued` |
| `CART_CONFLICT` | 409 | Cart is for another trip or class |
| `ACTIVE_HOLD_EXISTS` | 409 | Unpaid hold on this train; `details.pnr` |

## Holds, OTP and payment

| Code | HTTP | Meaning |
|---|---|---|
| `EMPTY_CART` | 409 | Locks expired before the hold |
| `PASSENGER_COUNT_MISMATCH` | 422 | Not one passenger per seat |
| `OTP_INVALID` | 422 | Wrong code; `details.attempts_left` |
| `OTP_EXPIRED` | 409 | Code expired |
| `OTP_LOCKED` | 429 | Too many attempts or sends; `details.retry_after` |
| `OTP_REQUIRED` | 409 | Pay only after the OTP is verified |
| `HOLD_EXPIRED` | 409 | Hold ran out |
| `ALREADY_PAID` | 409 | PNR already paid |

## Bookings

| Code | HTTP | Meaning |
|---|---|---|
| `NOT_CANCELLABLE` | 409 | Wrong status or train departed; `details.reason` |
| `TICKET_INVALID` | 422 | Ticket QR code is not genuine; `details.reason` |
