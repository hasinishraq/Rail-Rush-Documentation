# Internal endpoints

Every `cmd/app` pod also listens on an **internal port** (`:9091`; published locally for the auth
pod only). It must never be exposed to the internet.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/healthz` | — | Liveness: the process is up |
| `GET` | `/readyz` | — | Readiness: dependencies answer within 3 s (`503` with the error otherwise) |
| `GET` | `/metrics` | — | Prometheus metrics |
| `POST` | `/internal/v1/journeys/{id}/{warm\|open\|close}` | `X-Internal-Token` | Produce a Warm / Open / Close command now, exactly as the scheduler would → `202 {request_id, command}` |
| `GET` | `/internal/v1/journeys/{id}/invariants` | `X-Internal-Token` | Run the invariant checks for one journey |

`open` takes an optional body naming pools; without one it opens every pool whose `opens_at` has passed:

```sh
curl -X POST -H "X-Internal-Token: $INTERNAL_TOKEN" \
  -d '{"pools":[{"class_code":"S_CHAIR","quota":"ONLINE"}]}' \
  localhost:9091/internal/v1/journeys/74220261021/open

curl -H "X-Internal-Token: $INTERNAL_TOKEN" \
  localhost:9091/internal/v1/journeys/74220261021/invariants
```

The token is compared in constant time. When `INTERNAL_TOKEN` is empty the endpoints refuse every call.
