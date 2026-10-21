# Troubleshooting

## `required variable ... is missing a value`

`deploy/.env` is missing or a required secret is empty. See
[Configure secrets](../getting-started/local-setup.md#1-configure-secrets).

## Services fail with `password authentication failed`

The Postgres volume was initialised with different credentials than those now in `deploy/.env`.
Restore the previous values or reset the volume (deletes all data):

```sh
docker compose -p railway-application -f deploy/docker-compose.yml down -v
docker compose -p railway-application -f deploy/docker-compose.yml up -d
```

## The web app shows a blank page or an error box

Open the browser console for the stack trace. A session saved from an older stack (signed with
different keys) causes failures: sign out, or clear the site data for `localhost:8088`.

## `http://localhost:8088` hangs, but `http://127.0.0.1:8088` works

On Windows with Docker Desktop (WSL 2), the IPv6 localhost relay can stop forwarding after containers
are recreated. Use `127.0.0.1`, or restart Docker Desktop and the stack:

```sh
docker desktop restart
docker compose -p railway-application -f deploy/docker-compose.yml up -d
```

Postgres, Redis and Redpanda have no restart policy, so `up -d` is needed after a Docker restart.

## `migrate` fails with `relation … does not exist`

The Postgres volume holds an older schema. Reset it (deletes all data):

```sh
docker compose -p railway-application -f deploy/docker-compose.yml down -v
docker compose -p railway-application -f deploy/docker-compose.yml up -d
```

## Search shows no trains

Journeys exist for 12 days from the last `migrate` run. Run it again:

```sh
docker compose -p railway-application -f deploy/docker-compose.yml run --rm migrate
```

## "Sales haven't opened yet"

Pools open at their real times (08:00 or 14:00, 10 days ahead). For testing, start with
`DEV_OPEN_IN=1m`, or open a journey now:

```sh
curl -X POST -H "X-Internal-Token: $INTERNAL_TOKEN" localhost:9091/internal/v1/journeys/{id}/open
```

## Payment succeeded but the booking stays `PENDING_PAYMENT`

1. Check `outbox_oldest_unsent_seconds`: is the `relay` running?
2. Check the `allocator` and `projector` logs and consumer lag in Redpanda Console.
3. The fake gateway skips the webhook for some orders (`UNKNOWN_RATE`); the `reconciler` picks them up.

## Finding the OTP

Enter your `OTP_DEV_CODE` (if set in `deploy/.env`), or read the real code from the fake SMS gateway:

```sh
docker compose -p railway-application -f deploy/docker-compose.yml logs fakesms
curl 'localhost:8091/last?phone=%2B8801711111111'
```

## `429 OVERLOADED` or `503 LOG_UNAVAILABLE`

The pod is at `MAX_INFLIGHT`, a partition is lagging, or Kafka is down. No state was changed; retry
after `Retry-After`. Check `allocator_partition_lag_ms` and the Redpanda container.

## Ticket PDF shows `?` instead of characters

The embedded fonts cover Latin and Bangla only. Other scripts render as `?`.
