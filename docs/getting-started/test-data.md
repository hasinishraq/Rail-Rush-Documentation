# Test data and tools

## Users

The dev seed creates these accounts, all with the password `dev-password-123`:

| Phone | Name |
|---|---|
| `+8801700000001` | Dev user |
| `+8801711111111` | Ayesha Rahman |
| `+8801811111112` | Karim Uddin |
| `+8801911111113` | Nusrat Jahan |
| `+8801611111114` | Tanvir Hasan |
| `+8801511111115` | Farhana Akter |

The five named users have saved passengers. To create bookings for them through the API (confirmed,
partly cancelled and cancelled with refunds, over the next 5 days):

```sh
go run ./test/seeddata
```

## OTP and payments

- **OTP:** `OTP_DEV_CODE` from `deploy/.env` (if you set one) verifies any booking. The real
  code is sent to the fake SMS gateway: `curl 'localhost:8091/last?phone=%2B8801711111111'`.
- **Payment:** the web app redirects to the fake gateway's pay page. To settle an order from a script:
  `POST localhost:8090/orders/{order_id}/pay?outcome=success|fail|unknown`.

!!! danger
    Never set `OTP_DEV_CODE` outside local development. The API logs a warning at start-up when it is set.

## Journey IDs

A journey ID is `train number × 10⁸ + YYYYMMDD`. For example, `81620261022` is train 816
(Parjotak Express) on 22 October 2026.

## Test programs

| Command | What it checks |
|---|---|
| `go test ./...` | Inventory unit tests and a property test (3,000 random command sequences: no segment claimed twice, carts within limits, replay rebuilds identical state) |
| `go run ./test/smoke --from DHK --to CTG --class SNIGDHA` | Full user flow with two users: lock, conflicts, hold, OTP, payment, ticket, partial cancel, refund |
| `go run ./test/contention --journey 81620261022 --n 20000 --seats 64` | 20,000 clicks on 64 seats straight into Kafka: exactly one owner per seat |
| `go run ./test/saga --journey 81620261022` | Timers: lock expiry, hold expiry, late payment (needs short timers, see below) |

`test/smoke` reads `INTERNAL_TOKEN` from the environment (see [Configure secrets](local-setup.md#1-configure-secrets)).
Timer workflows need short TTLs:

```sh
HOLD_TTL=45s SELECT_IDLE_TTL=10s docker compose -p railway-application -f deploy/docker-compose.yml up -d allocator
go run ./test/saga --journey 81620261022
```

## Load testing

[k6](https://k6.io) scripts in `test/load` simulate a sales opening: signed-in viewers polling the seat
map and admitted users locking, holding and verifying seats (about 1,000 requests per second by default).
`test/loadprep` creates the users directly in Postgres and signs their tokens with the stack's keys,
because API login is intentionally slow and rate-limited. Commands are in `test/load/README.md`.

| Script | Scenario | Reference result (one laptop) |
|---|---|---|
| `opening.js` | Sales opening, 60 s | 911 req/s, 0 failures, seat lock p95 30 ms, seat map p95 7 ms, no double sale |
| `few-seats.js` | 1,000 users, 10 seats, same instant | Exactly 10 bookings; 94 % of losing clicks rejected by the seat mirror |

Check the invariants of a journey:

```sh
curl -H "X-Internal-Token: $INTERNAL_TOKEN" localhost:9091/internal/v1/journeys/81620261022/invariants
```
