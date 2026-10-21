# Getting started

The whole platform runs in Docker: the Go services, Postgres, Redis, Redpanda (Kafka), Traefik, the
web app, a fake payment gateway, a fake SMS gateway, Prometheus, Grafana and Loki.

## Prerequisites

| Tool | Version | Required for |
|---|---|---|
| Docker Desktop or Docker Engine + Compose v2 | Current | Running the stack |
| OpenSSL | Any | Generating secrets |
| Go | 1.26+ | Test tools in `test/` (optional) |
| Node.js | 22+ | Web app development outside Docker (optional) |

## Steps

1. [Run locally](local-setup.md): configure secrets, start the stack and open the app.
2. [Test data and tools](test-data.md): seeded users, the development OTP and the test programs.
