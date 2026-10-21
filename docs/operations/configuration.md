# Configuration

All services read environment variables (`internal/platform/config/config.go`). Durations use Go
syntax (`250ms`, `2m`, `720h`). Defaults apply when a variable is unset.

## Connections

| Variable | Default | Purpose |
|---|---|---|
| `PG_URL` | **required** | Postgres primary |
| `PG_READ_URL` | `PG_URL` | Read replica for queries |
| `REDIS_ADDR` | `redis:6379` | Redis |
| `KAFKA_BROKERS` | — | Kafka / Redpanda brokers |
| `ROLE` / `--role` | — | `api`, `live`, `auth`, `search`, `projector`, `relay`, `reconciler`, `scheduler`, `all` |

## Seat locks and holds

| Variable | Default | Purpose |
|---|---|---|
| `SELECT_IDLE_TTL` | `2m` | Lock released after this long without a click |
| `SELECT_MAX_TTL` | `5m` | Lock released after this long in any case |
| `MAX_SEATS_PER_USER` | `4` | Seats per user per journey (pool value wins) |
| `DAILY_SEAT_LIMIT` | `4` | Seats per user per travel day |
| `HOLD_TTL` | `10m` | Time for OTP and payment |
| `LATE_PAY_GRACE` | `30s` | Payments this late after hold expiry are still honoured |
| `OUTAGE_EXTEND_AFTER` | `30s` | Allocator outage after which deadlines are extended |

## Allocator and real time

| Variable | Default | Purpose |
|---|---|---|
| `SNAPSHOT_EVERY` | `5000` | Snapshot after this many events |
| `SNAPSHOT_INTERVAL` | `60s` | … or after this long |
| `LIVE_DELTA_INTERVAL` | `250ms` | Delta coalescing window |
| `SEATMAP_PUBLISH_INTERVAL` | `1s` | Seat-map snapshot publishing |
| `SMLOG_MAXLEN` | `2000` | Deltas kept for resume |
| `IDLE_EVICT_AFTER` | `30m` | Evict idle journeys from memory |
| `LIVE_MAX_CONNS` | `50000` | SSE connections per live pod |
| `LIVE_CONN_MAX_AGE` | `15m` | SSE connections are recycled after this |

## API behaviour

| Variable | Default | Purpose |
|---|---|---|
| `RESULT_WAIT` | `2s` | Wait for the allocator before answering `202` |
| `LAG_REJECT_MS` | `2000` | Reject early when a partition lags more than this |
| `MAX_INFLIGHT` | `2000` | Concurrent requests per pod before `429 OVERLOADED` |
| `RETURN_URL_ALLOWLIST` | `http://localhost,https://localhost` | Allowed payment `return_url` prefixes |
| `TRUST_PROXY` | `true` | Take the client IP from `X-Forwarded-For` |

## OTP and SMS

| Variable | Default | Purpose |
|---|---|---|
| `OTP_KEY` | **required for `api`** | HMAC key for OTP codes |
| `OTP_TTL` | `3m` | Code lifetime |
| `OTP_MAX_ATTEMPTS` | `5` | Wrong codes before lock |
| `OTP_MAX_RESENDS` | `3` | Resends per PNR |
| `OTP_DEV_CODE` | empty | **Local only.** A code accepted for every PNR |
| `NID_KEY` | `OTP_KEY` | HMAC key for national IDs |
| `SMS_PROVIDERS` | empty | SMS gateway URLs, primary first |

## Waiting room

| Variable | Default | Purpose |
|---|---|---|
| `ADMISSION_MODE` | `vendor` | `vendor` (verify only) or `local` (also issue tokens) |
| `ADMISSION_PUBKEY` | — | Vendor's public key (PEM or path) |
| `ADMISSION_KEY_FILE` | — | Local mode: private key that signs tokens |
| `ADMISSION_RATE` | `200` | Local mode: admissions per second |
| `ADMISSION_QUEUE_URL` | `/queue/status` | Returned in `ADMISSION_REQUIRED` errors |

## Authentication

| Variable | Default | Purpose |
|---|---|---|
| `JWT_SIGNING_KEY_FILE` | — | ES256 private key (auth role) |
| `JWT_KID` | `dev1` | Key ID in the JWT header and JWKS |
| `JWKS_URL` | — | Where non-auth pods fetch public keys |
| `ACCESS_TTL` | `15m` | Access token lifetime |
| `REFRESH_TTL` | `720h` | Refresh token lifetime (30 days) |
| `ARGON2_MEMORY_KIB` | `65536` | Argon2id memory |
| `ARGON2_TIME` | `1` | Argon2id iterations |
| `ARGON2_THREADS` | `2` | Argon2id parallelism |
| `TURNSTILE_SITE_KEY` | **required for `auth`** | Cloudflare Turnstile site key, served to the web app by `GET /v1/auth/captcha` |
| `TURNSTILE_SECRET_KEY` | **required for `auth`** | Turnstile secret used to verify the token that register and login require |

## Tickets

| Variable | Default | Purpose |
|---|---|---|
| `TICKET_SIGNING_KEY_FILE` | **required for `api`** | ES256 private key that signs the ticket QR code |
| `TICKET_KID` | `t1` | Key ID in the QR payload and `GET /v1/tickets/keys` |

## Payments

| Variable | Default | Purpose |
|---|---|---|
| `GATEWAYS` | `fake` | Enabled gateway adapters |
| `GATEWAY_BASE_URL` | `http://fakegateway:8090` | Server-to-server gateway URL |
| `GATEWAY_PUBLIC_URL` | `GATEWAY_BASE_URL` | Gateway URL as seen by browsers |
| `GATEWAY_SECRET` | — | Webhook signature secret |

## Other

| Variable | Default | Purpose |
|---|---|---|
| `INTERNAL_TOKEN` | empty | Required for `/internal/v1/*`; empty disables them |
| `LOG_LEVEL` | `info` | `debug`, `info`, `warn`, `error` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | empty | Reserved for tracing (not implemented) |

## Docker Compose

### Secrets

Read from `deploy/.env` (template: `deploy/.env.example`). Compose refuses to start while a required
variable is empty. See [Configure secrets](../getting-started/local-setup.md#1-configure-secrets).

| Variable | Required | Passed to |
|---|---|---|
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | Yes | Postgres; composed into `PG_URL` and `PG_READ_URL` |
| `GATEWAY_SECRET` | Yes | App `GATEWAY_SECRET`; fake gateway `SECRET` |
| `OTP_KEY`, `NID_KEY` | Yes | App |
| `INTERNAL_TOKEN` | Yes | App |
| `OTP_DEV_CODE` | No | App |
| `TURNSTILE_SITE_KEY`, `TURNSTILE_SECRET_KEY` | Yes | App |

### Overrides

| Variable | Default | Purpose |
|---|---|---|
| `DEV_OPEN_IN` | `0s` | Open sale windows this long after `migrate` runs |
| `HOLD_TTL` | `10m` | Hold lifetime, shortened for timer tests |
| `SELECT_IDLE_TTL` | `2m` | Idle lock lifetime, shortened for timer tests |
| `API_URL` | `http://localhost:8000` | Vite dev-server proxy target |
