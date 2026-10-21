# Redis keys

All key builders are in `internal/wire/wire.go`. The allocator is the main writer; the API, live and
search roles read.

## Seat maps and live updates

| Key / channel | Type | Writer | Readers | Purpose |
|---|---|---|---|---|
| `seatmap:{journey}` | string (JSON) | Allocator, ≤ 1/s | Search | Seat-map snapshot served by `GET /seatmap` |
| `segfree:{journey}:{class}:{quota}` | string | Allocator, every commit | API | Free seats per segment, for fast `SOLD_OUT` |
| `smd:{journey}` | pub/sub channel | Allocator, every 250 ms | Live, API seat mirror | Seat deltas |
| `smlog:{journey}` | capped stream / list (`SMLOG_MAXLEN`) | Allocator | Live | Resume after reconnect (`Last-Event-ID`) |
| `live:{user}:{journey}` | string | Live | Live | One stream per user and journey; a new one replaces the old |

## Commands and results

| Key / channel | Writer | Purpose |
|---|---|---|
| `req:{request_id}` | API | Owner of a pending request (so only that user can poll it) |
| `result:{request_id}` | Allocator | Decision for a command; TTL 15 min |
| `results:{pod}` | Allocator (publish) | Wakes the API pod waiting for the result |
| `cart:{journey}:{user}` | Allocator | The user's locked seats, trip and expiry |
| `hold:{pnr}` | Allocator | PNR state for OTP, payments and the booking view before the projection catches up; TTL 30 min |
| `lag:{partition}` | Allocator | Age of the newest processed command; API back-pressure; TTL 10 s |

## Limits and abuse

| Key | Purpose |
|---|---|
| `rl:{scope}:{id}:{window}` | Fixed-window rate-limit counters (per IP or per user) |
| `ulimit:{user}:{run_date}` | Seats issued per user per travel day (mirror of `purchase_counter`) |
| `taken:{user}:{journey}`, `taken:cool:{user}:{journey}` | Counts `SEAT_TAKEN` answers; 10 in 60 s give a 5 s cool-down |
| `otp:rl:{user}` | OTP sends per user per hour (max 10) |

## Admission and auth

| Key / channel | Purpose |
|---|---|
| `admit:{jti}` | Binds an admission token to the first user who presents it |
| `admit:used:{jti}` | Marks a token as used for a booking (stores the PNR) |
| `admission:user:{user}` | Local issuer: the user's current token, reused while valid and unused |
| `admission:bucket`, `admission:waiting` | Local issuer's token bucket and waiting counter |
| `jwt:deny:{jti}` | Revoked access tokens until they expire |
| `jwt:deny` (channel) | Pushes revocations to every pod's in-memory Bloom filter |

## Idempotency

| Key | Purpose |
|---|---|
| `idem:{user}:{endpoint}:{key}` | Fast path for idempotent API calls (the durable copy is in `idempotency_key`) |

!!! note
    Redis holds no data that cannot be rebuilt. Carts, holds and seat maps are re-published by the
    allocator from its in-memory state, which itself is rebuilt from Kafka.
