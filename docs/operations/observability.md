# Observability

## Metrics

Every pod exposes Prometheus metrics on `/metrics` (internal port). Prometheus (`:9090`) scrapes them
with `deploy/prometheus.yml`; Grafana (`:3000`) opens the `railway` dashboard by default.

| Metric | Type | Watch for |
|---|---|---|
| `http_requests_total{route,status}` | counter | Error rates per route |
| `http_request_duration_seconds{route}` | histogram | Latency per route |
| `api_errors_total{code}` | counter | Spikes of `OVERLOADED`, `LOG_UNAVAILABLE`, `INTERNAL` |
| `command_results_total` | counter | Select/release/hold/cancel outcomes; `FAST_*` = answered without Kafka |
| `command_decision_seconds` | histogram | Produce-to-result latency seen by the API |
| `allocator_commit_seconds` | histogram | Kafka transaction latency |
| `allocator_commit_failures_total` | counter | Any increase |
| `allocator_events_total{kind}` | counter | Event mix |
| `allocator_partition_lag_ms` | gauge | Sustained values near `LAG_REJECT_MS` |
| `allocator_resident_journeys` | gauge | Memory pressure |
| `live_connections` | gauge | Near `LIVE_MAX_CONNS` |
| `live_delta_age_seconds` | histogram | Deltas older than ~1 s at the live pod |
| `live_slow_disconnects_total` | counter | Clients too slow for the stream |
| `seatmirror_journeys` | gauge | Hot journeys mirrored per API pod |
| `otp_sent_total{kind}`, `otp_verify_total{result}` | counter | OTP abuse, delivery problems |
| `sms_sent_total{provider,outcome}` | counter | SMS provider failures |
| `outbox_oldest_unsent_seconds` | gauge | Relay falling behind (payments not confirming) |
| `projector_events_total{kind}` | counter | Projection progress |
| `projector_exclude_violations_total` | counter | **Page immediately**: a double-sell was blocked |
| `payments_initiated_stuck` | gauge | Payments `INITIATED` for over 15 minutes |
| `refunds_not_succeeded` | gauge | Refunds waiting or failing |
| `webhook_signature_failures_total` | counter | Forged or misconfigured webhooks |
| `auth_refresh_reused_total` | counter | Possible token theft |

## Logs

Services log structured JSON (`log/slog`) to stdout with the `role` and `pod`. Every request carries
an `X-Request-Id`, echoed in responses and error bodies, which ties a user report to the logs.

Lines to alert on:

| Text | Meaning |
|---|---|
| `ALERT payment amount mismatch` | A capture did not match the charged amount; refunded |
| `ALERT second capture for a PNR; refunding` | Duplicate capture refunded |
| `webhook signature failure` | Bad gateway signature |
| `OTP_DEV_CODE is set` | The dev OTP is enabled: must never appear in production |

## Invariant checks

Run after every sale window (and on demand):

```sh
curl -H "X-Internal-Token: $INTERNAL_TOKEN" localhost:9091/internal/v1/journeys/{id}/invariants
```

Any reported violation (overlapping claims, a seat map ahead of the log, tickets that differ from the
log, money without a ticket, a purchase-limit breach) is a bug to investigate immediately. See
[Seat allocator → Invariant checks](../architecture/allocator.md#invariant-checks).

## Kafka

Redpanda Console (`:8081`) shows topics, partitions, consumer-group lag and individual messages
(protobuf payloads appear as bytes).

## Not implemented

OpenTelemetry tracing (`OTEL_EXPORTER_OTLP_ENDPOINT` is reserved) and log archiving.
