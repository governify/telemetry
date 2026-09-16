# Telemetry backend

Observability stack (Caddy + OTel Collector + Tempo + Prometheus + Loki + Grafana) fronting
`telemetry.governify.io`.

Exposed via `governify/nginx-streamer` (TLS SNI passthrough on 443, plain HTTP on 80), which
forwards `*.telemetry.governify.io` to the `telemetry-caddy` container here. Caddy terminates TLS
(automatic Let's Encrypt certs) and reverse-proxies to the right internal service.

## What's here

- `telemetry.governify.io` → landing page (`landing/index.html`)
- `grafana.telemetry.governify.io` → Grafana, provisioned with Prometheus, Tempo and Loki as
  datasources
- `collector.telemetry.governify.io` → OTel Collector's zpages debug page
- Tempo, Prometheus, Loki: **internal only**, no subdomain — queried by Grafana over the internal
  `telemetry` docker network
- OTel Collector's OTLP receiver (4317 gRPC / 4318 HTTP): published directly on the host,
  **unauthenticated over plain HTTP** — this is how instrumented apps on other servers send data
  in today. Known gap, not fronted by Caddy/TLS.

Dashboards live in `dashboards/*.json` and are auto-provisioned into Grafana on startup — drop a
JSON file in and recreate the `grafana` service to pick it up.

## Quickstart

The `telemetry` docker network is shared with `nginx-streamer`, so it must exist first:

```bash
docker network create telemetry
```

Then bring the stack up:

```bash
docker-compose -f telemetry-docker-compose.yaml up -d
```

And (re)start `nginx-streamer` so it joins the `telemetry` network too — see its own README.

Config under `config/` and `dashboards/` is bind-mounted. After editing a file,
`docker-compose up -d` alone often won't pick it up — force it:

```bash
docker-compose -f telemetry-docker-compose.yaml up -d --force-recreate <service>
```

