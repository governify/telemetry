# Telemetry

Observability for `telemetry.governify.io`: OpenTelemetry → Prometheus / Tempo / Loki → Grafana.

Two parts:

- [`generators/`](generators/README.md): what produces data. **Install an agent on each server, instrument each app.**
- [`gateway-and-backends/`](gateway-and-backends/README.md): where the data goes. Collector, backends, Grafana, dashboards.
