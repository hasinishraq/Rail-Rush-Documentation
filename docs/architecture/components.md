# Components and roles

The backend is one Go module. `cmd/app` is a single binary started with `--role`; each role is a
separate container in Docker Compose (and a separate deployment in production). The allocator is a
separate binary because it is stateful.

## Roles of `cmd/app`

| Role | Serves | Owns / does | Scales |
|---|---|---|---|
| `auth` | `/v1/auth/*`, `/.well-known/jwks.json`; internal endpoints on `:9091` | Registration, login, refresh-token rotation, logout, JWT signing | Horizontally |
| `search` | `/v1/stations`, `/v1/search`, `/v1/journeys/{id}`, `/v1/layouts/{id}`, `/v1/journeys/{id}/seatmap` | Read-only reference data and seat-map snapshots | Horizontally (CDN-friendly) |
| `api` | Selection, holds, OTP, payments, bookings, tickets and ticket verification, profile, admission (local), webhooks | Produces commands, waits for results, owns OTP/payment/booking reads | Horizontally (3 replicas locally) |
| `live` | `/v1/journeys/{id}/seatmap/live` (SSE) | Fan-out of seat deltas to every signed-in viewer | Horizontally, capped by `LIVE_MAX_CONNS` per pod |
| `projector` | — | `inventory-events` → `booking`, `ticket`, `purchase_counter`, ticket SMS | One consumer group |
| `relay` | — | Outbox → Kafka; `notifications` → SMS gateways | One instance (or more; rows are claimed) |
| `reconciler` | — | Finds stuck payments by asking the gateway; issues refunds | One instance |
| `scheduler` | — | Warm / Open / Close commands at the right time; monthly partitions | Leader-elected (`pg_try_advisory_lock(42)`) |

Other commands of the same binary:

| Command | What it does |
|---|---|
| `app migrate up [--seed=dev] [--open-in=2m]` | Applies `migrations/` (goose) and optionally loads the Bangladesh dev seed |
| `app keygen --dir /keys` | Creates the ES256 keys for JWTs, admission tokens and ticket QR codes if missing |
| `app invariants` | Runs the invariant checks from the command line |

## Allocator (`cmd/allocator`)

The stateful inventory owner. It joins the `booking-requests` consumer group with a transactional
session; each owned partition gets one worker goroutine that holds the journeys of that partition in
memory. See [Seat allocator](allocator.md).

Three replicas run locally. When one fails, Kafka rebalances its partitions to the others, which
rebuild the affected journeys from snapshot and log. `test/contention` verifies that no seat is
locked twice when an allocator is killed mid-run.

## Edge

| Component | Role |
|---|---|
| **nginx** (`web` container, `:8088`) | Serves the built web app with SPA fallback; proxies `/v1/` and `/.well-known/` to Traefik with buffering off (for SSE) |
| **Traefik** (`:8000`) | Routes by path to the roles using Docker labels. Explicit priorities make `/seatmap/live` go to `live` and selection/holds/payments/bookings/me/admission go to `api`, ahead of the `search` role's `/v1/journeys` prefix |

## Infrastructure

| Service | Used for |
|---|---|
| PostgreSQL 16 | Reference data, users, refresh tokens, OTPs, payments, refunds, outbox, idempotency keys, bookings and tickets (projection), snapshots |
| Redis 7 | Seat maps, deltas (pub/sub + capped log), carts, holds, command results, admission bindings, rate limits, JWT denylist |
| Redpanda (Kafka API) | `booking-requests`, `inventory-events`, `payment-events`, `notifications` |
| Fake gateway / fake SMS | Local stand-ins for the payment provider and the SMS providers |
| Prometheus + Grafana | Metrics and the `railway` dashboard |

## Source layout

```text
cmd/app            monolith: --role=api|live|auth|search|projector|relay|reconciler|scheduler|all
cmd/allocator      stateful inventory owner
cmd/fakegateway    local payment gateway with delay / failure / unknown-status rates
cmd/fakesms        local SMS gateway
proto/booking/v1   commands.proto, events.proto, snapshot.proto
internal/inventory pure core: masks, claims, carts, holds, Decide/Apply, timers, snapshots, invariants
internal/allocator partition loop, Kafka transactions, results, seat maps, deltas, snapshots
internal/api       router, middleware, handlers, DTOs, idempotency, PDF ticket
internal/live      SSE hub
internal/seatmirror in-memory seat maps of hot journeys for fast rejections
internal/auth      argon2id, ES256 JWT, refresh tokens with reuse detection, JWKS, denylist, NID HMAC
internal/admission admission-token verifier + local token-bucket issuer
internal/otp       6-digit codes, HMAC-stored
internal/notify    notifications consumer → SMS gateways
internal/search    stations, search, journey, layout, seat map read
internal/booking   command producer, result waiter, Redis views, PNRs
internal/payment   gateway adapter, webhooks, refunder, reconciler
internal/outbox    transactional outbox + relay
internal/projector inventory-events → Postgres projection
internal/scheduler Warm / Open / Close, partitions
internal/catalog   in-process reference data
internal/fares     fare chart, service charge, child fare, refund rules
migrations/        schema (goose) and dev seed
deploy/            docker-compose.yml, Dockerfiles, Prometheus, Grafana
test/              smoke, contention, saga, seeddata
web/               React + TypeScript + Vite app
```
