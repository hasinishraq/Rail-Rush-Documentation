# API overview

The public API is JSON over HTTPS under `/v1`. Locally it is reachable on <http://localhost:8000>
(Traefik) or, same-origin with the web app, on <http://localhost:8088/v1>.

## Endpoints

| Method | Path | Auth | Page |
|---|---|---|---|
| `POST` | `/v1/auth/register` | — + `Idempotency-Key` | [Authentication](auth.md) |
| `POST` | `/v1/auth/login` | — | [Authentication](auth.md) |
| `POST` | `/v1/auth/refresh` | — | [Authentication](auth.md) |
| `POST` | `/v1/auth/logout` | User | [Authentication](auth.md) |
| `GET` | `/v1/auth/captcha` | — | [Authentication](auth.md#captcha) |
| `GET` | `/.well-known/jwks.json` | — | [Authentication](auth.md) |
| `GET` | `/v1/me` | User | [Authentication](auth.md) |
| `GET` `POST` | `/v1/me/passengers` | User (+ key on POST) | [Authentication](auth.md) |
| `DELETE` | `/v1/me/passengers/{id}` | User | [Authentication](auth.md) |
| `GET` | `/v1/stations` | — | [Search](search.md) |
| `GET` | `/v1/search` | — | [Search](search.md) |
| `GET` | `/v1/journeys/{id}` | — | [Search](search.md) |
| `GET` | `/v1/layouts/{layoutId}` | — | [Search](search.md) |
| `GET` | `/v1/journeys/{id}/seatmap` | User | [Search](search.md) |
| `POST` | `/v1/admission` | User (local mode) | [Selection](selection.md) |
| `GET` | `/v1/journeys/{id}/seatmap/live` | User | [Selection](selection.md#live-seat-updates) |
| `PUT` `DELETE` | `/v1/journeys/{id}/selection/{seatId}` | User + admission | [Selection](selection.md) |
| `GET` | `/v1/journeys/{id}/selection` | User | [Selection](selection.md) |
| `GET` | `/v1/requests/{requestId}` | User | [Selection](selection.md#asynchronous-results) |
| `GET` | `/v1/requests/{requestId}/stream` | User | [Selection](selection.md#asynchronous-results) |
| `POST` | `/v1/holds` | User + admission + `Idempotency-Key` | [Holds and OTP](holds.md) |
| `POST` | `/v1/holds/{pnr}/otp/verify` | User | [Holds and OTP](holds.md) |
| `POST` | `/v1/holds/{pnr}/otp/resend` | User | [Holds and OTP](holds.md) |
| `POST` | `/v1/payments` | User + `Idempotency-Key` | [Payments](payments.md) |
| `GET` | `/v1/payments/{paymentId}` | User | [Payments](payments.md) |
| `POST` | `/v1/webhooks/payment/{gateway}` | Gateway signature | [Payments](payments.md) |
| `GET` | `/v1/me/bookings` | User | [Bookings](bookings.md) |
| `GET` | `/v1/bookings/{pnr}` | User | [Bookings](bookings.md) |
| `POST` | `/v1/bookings/{pnr}/cancel` | User + `Idempotency-Key` | [Bookings](bookings.md) |
| `GET` | `/v1/bookings/{pnr}/ticket` | User | [Bookings](bookings.md) |
| `GET` | `/v1/tickets/keys` | — | [Bookings](bookings.md#public-keys) |
| `POST` | `/v1/tickets/verify` | Staff | [Bookings](bookings.md#verify-a-ticket) |

Operator endpoints on the internal port are in [Internal endpoints](internal.md).

## Conventions

### Authentication

Send the access token from login as a bearer token:

```http
Authorization: Bearer eyJhbGciOiJFUzI1NiIsImtpZCI6ImRldjEi...
```

An expired token gives `401 TOKEN_EXPIRED`: call `/v1/auth/refresh` and retry. Seat selection and
holds also need an `X-Admission-Token` (see [Waiting room](selection.md#waiting-room)); watching the
live seat map does not.

### Errors

Every error has the same envelope and a stable, machine-readable `code`:

```json
{
  "error": {
    "code": "SEAT_TAKEN",
    "message": "seat is taken",
    "request_id": "01a1592c-f9bc-7795-9444-7939643c809a",
    "details": { "seat_id": "CHA-10" }
  }
}
```

Branch on `code`, never on `message`. `429` and `503` responses carry `Retry-After`. The full list is
in [Error codes](errors.md).

### Idempotency

Calls that create something take an `Idempotency-Key` header with a **UUID**. Retrying with the same
key returns the original response; reusing a key with a different body gives
`422 IDEMPOTENCY_KEY_REUSED`; a retry while the first call is still running gives
`409 REQUEST_IN_PROGRESS`.

### Asynchronous results

Commands decided by the allocator (select, release, hold, cancel) usually answer within milliseconds.
If no decision arrives within 2 s, the API returns `202 Accepted`:

```json
{
  "request_id": "01a1595d-d6b9-7a69-a3a3-86d92fe0e38a",
  "poll_url": "/v1/requests/01a1595d-d6b9-7a69-a3a3-86d92fe0e38a",
  "stream_url": "/v1/requests/01a1595d-d6b9-7a69-a3a3-86d92fe0e38a/stream"
}
```

Poll or stream it to get the final status code and body. See [Asynchronous results](selection.md#asynchronous-results).

### Tracing

Every response has an `X-Request-Id` header (generated unless the client sent one, up to 128
characters). It is the `request_id` in error bodies and in the server logs.

### Money and time

Money is `{ "amount_minor": 76500, "currency": "BDT" }` (minor units: 76500 = Tk 765.00). Times are
RFC 3339 in UTC; display them in `Asia/Dhaka`. Dates are `YYYY-MM-DD`.

### Rate limits

| Scope | Limit |
|---|---|
| Register, login (per IP) | 20 / min |
| Refresh (per IP) | 60 / min |
| Search (per IP) | 60 / min |
| Stations, journey, layout (per IP) | 120 / min |
| Seat clicks (per user and journey) | 30 / min, plus a 5 s cool-down after 10 `SEAT_TAKEN` in 60 s |
| Holds (per user) | 5 / min |
| Payments (per user) | 10 / min |
| OTP sends (per user) | 10 / hour |

Exceeding a limit gives `429 RATE_LIMITED` with `details.retry_after` and `Retry-After`.
