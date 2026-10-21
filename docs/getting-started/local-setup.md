# Run locally

All commands run from the repository root. The examples use this shorthand:

```sh
alias dc='docker compose -p railway-application -f deploy/docker-compose.yml'
```

## 1. Configure secrets

Compose reads credentials from `deploy/.env`, which is excluded from git and from Docker build
contexts. Create it from the template:

```sh
cp deploy/.env.example deploy/.env
```

Set a random value for each empty variable:

```sh
openssl rand -hex 32
```

| Variable | Purpose |
|---|---|
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | Postgres account and database |
| `GATEWAY_SECRET` | Payment webhook signatures (app and fake gateway) |
| `OTP_KEY`, `NID_KEY` | HMAC keys for OTP codes and national IDs |
| `INTERNAL_TOKEN` | `X-Internal-Token` for the internal endpoints |
| `OTP_DEV_CODE` | Optional 6-digit development OTP accepted for every booking (empty in the template; you choose it) |

Compose refuses to start while a required variable is empty. To use the values in a shell session
(internal endpoints, smoke test, load test):

```sh
set -a; . deploy/.env; set +a
```

!!! warning
    Postgres keeps the credentials it was initialised with, and existing national ID hashes depend on
    `NID_KEY`. After changing `POSTGRES_*` or `NID_KEY`, [reset the stack](#reset).

## 2. Start the stack

```sh
dc up -d --build
```

The `migrate` job creates the schema and loads the Bangladesh seed: 24 stations, 22 intercity trains
on five corridors from Dhaka, and journeys for today and the next 11 days.

To open the sale windows shortly after start instead of at their scheduled times:

```sh
DEV_OPEN_IN=1m dc up -d --build
```

!!! tip "Limited resources"
    Add `--scale api=1 --scale live=1 --scale allocator=2` and omit `console prometheus grafana`.

## Endpoints

| URL | Service |
|---|---|
| <http://localhost:8088> | Web app (nginx; proxies `/v1` to Traefik) |
| <http://localhost:8000> | Public API (Traefik) |
| <http://localhost:9091> | Internal endpoints of the auth pod (`/internal/v1/...`, `/readyz`, `/metrics`) |
| <http://localhost:8090> | Fake payment gateway |
| <http://localhost:8091> | Fake SMS gateway (`GET /last?phone=`) |
| <http://localhost:8081> | Redpanda Console |
| <http://localhost:9090> | Prometheus |
| <http://localhost:3000> | Grafana (anonymous viewer) |

## Rebuild after code changes

```sh
dc build keygen      # Go services (all roles share the railway-app image)
dc build web         # Web app
dc up -d
```

## Reset

Deletes the Postgres volume and the signing keys: all users, bookings and sessions are lost.

```sh
dc down -v
dc up -d --build
```

## Web app development

```sh
cd web
npm ci
npm run dev          # http://localhost:5173, proxies /v1 to http://localhost:8000
```

Set `API_URL` to proxy to a different API.

## Extend the journey calendar

The seed covers 12 days from the last `migrate` run. To add journeys for later days:

```sh
dc run --rm migrate
```
