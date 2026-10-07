# Gateway and backends

The central stack: it receives the telemetry sent by the [generators](../generators/README.md), stores it and shows it in Grafana.

```
agents / apps ──OTLP──▶ Collector ──▶ Prometheus  (metrics)
                        (4317/4318)├─▶ Tempo + Jaeger (traces)
                                   └─▶ Loki  (logs)
                                         ▲
                       Grafana ──────────┘  reads all of them
```

## What runs

| Service | What it does | Reachable at |
|---|---|---|
| Caddy | HTTPS (automatic Let's Encrypt) in front of the web UIs | — |
| Collector | Receives OTLP and forwards it to the backends | `:4317` gRPC / `:4318` HTTP on the host |
| Prometheus | Metrics + the recording and alert rules | internal |
| Tempo | Traces (also derives span metrics) | internal |
| Jaeger | A second UI for traces | `jaeger.telemetry.governify.io` |
| Loki | Logs | internal |
| Grafana | Dashboards over all of the above | `grafana.telemetry.governify.io` |

`telemetry.governify.io` is a landing page with links; `collector.telemetry.governify.io` is the Collector's debug page.
Internal means only Grafana can reach it.

On this server `governify/nginx-streamer` receives port 443 and forwards `*.telemetry.governify.io` to Caddy.

## Start it

Needs Docker and docker-compose. From this folder:

```bash
cp .env.example .env     # set GRAFANA_ADMIN_USER and GRAFANA_ADMIN_PASSWORD
docker network create telemetry          # once; nginx-streamer shares it
docker-compose -f telemetry-docker-compose.yaml up -d
```

Open Grafana and log in with the user from `.env`. Data starts to appear when an agent is connected.

## Change something

| You edited | Run |
|---|---|
| A dashboard (`dashboards/*.json`) | Nothing: Grafana reloads it in ~10 s |
| Prometheus rules (`config/prometheus-rules/`) | `docker exec telemetry_prometheus_1 kill -HUP 1` |
| Any other file in `config/` | `docker-compose -f telemetry-docker-compose.yaml up -d --force-recreate <service>` |
| `telemetry-docker-compose.yaml` or `.env` | `docker-compose -f telemetry-docker-compose.yaml up -d` |

Plain `up -d` often misses edits to mounted files, so use `--force-recreate` for those. Image versions are pinned: bump them on purpose.

## Dashboards

| Dashboard | What it shows |
|---|---|
| `noc-overview` | Wall screen: servers, applications, alerts, health over time |
| `app-status` | One application: its services, requests, errors, logs |
| `server-status` | One server: CPU, memory, disk, containers |
| `service-deep-dive` | One service: traces, latency, errors, dependencies |
| `otel-collector-15983` | The Collector's own health (from grafana.com) |

Click-through: NOC → app → service, and NOC → server.

**To change one:** edit it in Grafana without saving, export it as JSON and paste it into `dashboards/<name>.json`.
Never keep two files with the same `uid` (`metadata.name`): it blocks every dashboard. Saving from the Grafana UI is
disabled; enabling it needs `allowUiUpdates: true` in `config/grafana-dashboards.yaml`, and then the change lives only in Grafana.

Panels read the recording rules in `config/prometheus-rules/`, not the raw metrics. The labels they depend on are
in the [label contract](../generators/README.md#label-contract).

## Alerts

The rules in `config/prometheus-rules/alerts.rules.yml` show up in Grafana as `ALERTS`. **Nothing is sent to anyone**: there is no Alertmanager.

## Wipe all data

Deletes every stored metric, trace and log. Grafana (dashboards, users) and Caddy (certificates) are kept. **Irreversible.**

```bash
docker-compose -f telemetry-docker-compose.yaml stop tempo jaeger prometheus loki
docker-compose -f telemetry-docker-compose.yaml rm -f tempo jaeger prometheus loki
docker volume rm telemetry_tempo-data telemetry_jaeger-data telemetry_prometheus-data telemetry_loki-data
docker-compose -f telemetry-docker-compose.yaml up -d
```

## Known gaps

- Ports 4317/4318 accept data **without TLS or authentication**: anyone who knows the address can send.
- No alert delivery (see Alerts). Retention: Prometheus 30 days, Tempo 14 days.
