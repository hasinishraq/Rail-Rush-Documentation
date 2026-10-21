# Security

Controls are grouped by authentication, authorization, booking and payment integrity, and abuse
prevention.

## Authentication

| Control | Implementation |
|---|---|
| Password storage | Argon2id with a random 16-byte salt per password, PHC string format; constant-time comparison (`internal/auth/password.go`) |
| Password policy | At least 10 characters, rejected if in the breached-password list |
| User enumeration | Unknown users still run a dummy Argon2 hash, so timing does not reveal registered numbers; every failure is `INVALID_CREDENTIALS` |
| Access tokens | ES256 JWTs, 15 minutes; the verifier accepts only ES256 and checks issuer, audience and expiry (no `alg=none` or algorithm confusion) |
| Key distribution | Public keys published at `/.well-known/jwks.json` with a `kid`, so keys can be rotated |
| Refresh tokens | Random, stored only as SHA-256, single-use and rotated on every refresh |
| Theft detection | Replaying a used refresh token revokes the whole token family (`REFRESH_REUSED`, `auth_refresh_reused_total`) |
| Logout / revocation | Revokes the refresh family and denylists the access token's `jti` in Redis; every pod follows the `jwt:deny` channel into an in-memory Bloom filter |
| National ID | Never stored: only HMAC-SHA256(NID) with a server key, used to enforce one account per NID |

## Authorization

| Control | Implementation |
|---|---|
| Authenticated routes | Everything except search, stations, journey, layout, register, login and refresh needs a valid bearer token |
| Ownership | Bookings, tickets, payments, carts and request results are checked against the token's user; staff roles (`counter`, `agent`, `admin`) are the only exception |
| No probing | Someone else's PNR or payment gives `404`, not `403`, so their existence is not revealed |
| Waiting room | Seat locks and holds need an ES256 admission token (watching the live seat map needs only a login), **bound to the first user who uses it** (not shareable) and **consumed by one booking** |
| Internal API | Separate port; operator endpoints need `X-Internal-Token`, compared in constant time; refused entirely when no token is configured |

## Booking and payment integrity

| Threat | Control |
|---|---|
| Double-selling a seat | Single writer per journey in the allocator, plus the Postgres `EXCLUDE` constraint on tickets as a second line of defence |
| Booking for someone else's phone | OTP to the account's phone before payment: stored as HMAC(PNR ‖ code), 3 minutes, 5 attempts, resend limits |
| Forged payment confirmation | Webhooks must carry a valid HMAC-SHA256 signature over the raw body; failures are logged and counted |
| Replayed webhook | `webhook_event (gateway, event_id)` primary key: each delivery is applied once |
| Amount tampering | A capture whose amount or currency differs from the payment is marked failed, alerted and refunded |
| Double charge | One live payment per PNR (unique index); a second capture is refunded automatically |
| Forged or altered tickets | The ticket QR payload is signed with ES256; `POST /v1/tickets/verify` checks the signature and the current seat status |
| Payment link reuse | `redirect_used_at`: the gateway link is single-use |
| Open redirect after payment | `return_url` must match `RETURN_URL_ALLOWLIST` |
| Duplicate submissions | `Idempotency-Key` (UUID) on register, holds, payments, cancellations and saved passengers; reuse with a different body is rejected |
| Lost money | `v_money_without_ticket` and the invariant checker find captured payments with no ticket and no refund; the reconciler resolves stuck payments |

## Abuse and availability

- Cloudflare Turnstile on register and login (when `TURNSTILE_SECRET_KEY` is set). The token is
  verified server-side and is single-use; if Cloudflare cannot be reached the request fails with
  `503 UNAVAILABLE` rather than skipping the check. Without Turnstile, five failed logins lock the
  account for 15 minutes.
- Per-IP rate limits on register, login, refresh and search; per-user limits on seat clicks, holds,
  payments and OTP sends. See [API overview](api/index.md#rate-limits).
- A 5-second cool-down after 10 `SEAT_TAKEN` answers in 60 s, against seat-sniping bots.
- The waiting room limits how fast users enter the booking flow.
- Per-pod in-flight request cap (`MAX_INFLIGHT`) and per-pod SSE cap (`LIVE_MAX_CONNS`) return `429`
  rather than collapse; lagging allocator partitions shed load early (`LAG_REJECT_MS`).

## Code-level hygiene

- **SQL:** every query uses pgx parameters; no SQL is built from user input.
- **Validation:** phone, NID, seat IDs, PNRs and Idempotency-Keys are checked against strict patterns; names are length-limited; enums are whitelisted. The database repeats key rules as `CHECK` constraints.
- **Errors:** panics are recovered per request and return a generic `INTERNAL` with a request ID; internal details are logged, not returned.
- **Containers:** the app runs as an unprivileged user (UID 10001) in a minimal Alpine image.
