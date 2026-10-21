# Docker deployment

`deploy/docker-compose.yml` runs the whole platform locally under the project name
`railway-application`. Secrets come from `deploy/.env`
(see [Configure secrets](../getting-started/local-setup.md#1-configure-secrets)).

```sh
cp deploy/.env.example deploy/.env    # first run only; fill in the values
docker compose -p railway-application -f deploy/docker-compose.yml up -d --build
```

## Images

| Image | Dockerfile | Contents |
|---|---|---|
| `railway-app:dev` | `deploy/app.Dockerfile` | `cmd/app`, all roles; Go build on `golang:1.26-alpine`, runs on `alpine:3.20` as UID 10001 |
| `railway-allocator:dev` | `deploy/allocator.Dockerfile` | `cmd/allocator` |
| `railway-web:dev` | `deploy/web.Dockerfile` | `npm ci && npm run build`, served by `nginx:1.27-alpine` with `web/nginx.conf` |
| `railway-fakegateway:dev` | `deploy/fakegateway.Dockerfile` | Local payment gateway |
| `railway-fakesms:dev` | `deploy/fakesms.Dockerfile` | Local SMS gateway |

## Services

| Service | Replicas | Notes |
|---|---|---|
| `keygen` | job | Creates ES256 keys (JWT, admission, ticket QR) in the `keys` volume |
| `migrate` | job | Schema + dev seed (`--seed=dev --open-in=$DEV_OPEN_IN`) |
| `redpanda-init` | job | Creates topics |
| `api` | 3 | Behind Traefik |
| `live` | 3 | SSE, unbuffered forwarding |
| `auth` | 1 | Publishes `9091` for internal endpoints |
| `search`, `projector`, `relay`, `reconciler`, `scheduler` | 1 each | |
| `allocator` | 3 | |
| `postgres`, `redis`, `redpanda`, `console` | 1 each | |
| `traefik` | 1 | `:8000` |
| `web` | 1 | `:8088` |
| `fakegateway`, `fakesms` | 1 each | `:8090`, `:8091` |
| `prometheus`, `grafana` | 1 each | `:9090`, `:3000` |
| `loki` | 1 | Central log store, 7-day retention, `:3100` |
| `alloy` | 1 | Ships every container's logs to Loki; UI on `:12345` |

Container logs are searchable in Grafana → **Railway Logs**, or in Explore with the Loki data source
(for example `{service="api", level="ERROR"}`).

Start-up order: app services wait for `keygen`, `migrate` and `redpanda-init` to complete and for
Redis to start; `migrate` waits for Postgres to be healthy.

## Common tasks

=== "Rebuild after Go changes"

    ```sh
    docker compose -p railway-application -f deploy/docker-compose.yml build keygen allocator
    docker compose -p railway-application -f deploy/docker-compose.yml up -d
    ```

=== "Rebuild the web app"

    ```sh
    docker compose -p railway-application -f deploy/docker-compose.yml build web
    docker compose -p railway-application -f deploy/docker-compose.yml up -d web
    ```

=== "Clean rebuild"

    ```sh
    docker compose -p railway-application -f deploy/docker-compose.yml build --no-cache
    docker compose -p railway-application -f deploy/docker-compose.yml up -d
    docker image prune -f
    ```

=== "Reset all data"

    ```sh
    docker compose -p railway-application -f deploy/docker-compose.yml down -v
    docker compose -p railway-application -f deploy/docker-compose.yml up -d
    ```

=== "Logs"

    ```sh
    docker compose -p railway-application -f deploy/docker-compose.yml logs -f api allocator
    ```

## Production notes

The Compose stack stands in for the production platform (Kubernetes, managed Postgres, Redis and
Kafka, CDN, waiting-room vendor). For production:

- Load secrets and keys from a secret manager, not from `deploy/.env`; leave `OTP_DEV_CODE` unset.
- Do not publish database, cache, broker or internal ports.
- Terminate TLS at the edge and add security headers.
- Run `ADMISSION_MODE=vendor` with the vendor's public key.
- Replace the fake gateway and SMS adapters.
