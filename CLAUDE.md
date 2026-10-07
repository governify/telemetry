# Telemetry stack — working rules

Hand-maintained observability stack (Caddy + OTel Collector + Tempo + Prometheus + Loki + Grafana)
fronting `telemetry.governify.io`. Layout: `gateway-and-backends/` (the central stack — compose, config,
dashboards, landing) and `generators/` (one folder per thing that sends telemetry; install docs live in `generators/README.md`). Keep
`gateway-and-backends/landing/index.html` and `gateway-and-backends/README.md` in sync — never let
them go stale. No boilerplate.

## Sync rule

Any change to `gateway-and-backends/telemetry-docker-compose.yaml` or `gateway-and-backends/config/Caddyfile` that adds, removes, exposes,
or un-exposes a service **must** update both files in the same change, not as a follow-up:

- New Caddy subdomain → add an entry to `gateway-and-backends/landing/index.html` (same style as the existing entries,
  `target="_blank" rel="noopener"`).
- Service made internal-only → remove its landing page entry or mark it `class="disabled"` (see
  the Tempo/Prometheus/Loki entries for the pattern) — never leave a dead link.
- New/removed compose service → update `gateway-and-backends/README.md`'s service list.

Don't ask before doing this — it's expected as part of finishing the change.

## Style notes (from prior feedback in this project)

- Landing page stays minimal: title + a plain link list. No footer, no status badges, no
  subtitle — explicitly rejected before as looking AI-generated.
- After editing a bind-mounted config file, prefer `up -d --force-recreate <service>` over
  `reload`/`restart` alone — single-file bind mounts here go stale because the editor replaces
  the file (new inode) rather than writing in place. Verify with
  `diff <(docker exec <container> cat <path>) <local path>` (distroless images — collector, tempo — have
  no `cat`: use `docker cp <container>:<path> - | tar -xO`). Run compose from `gateway-and-backends/`.
- Don't expose a service externally unless explicitly asked — Tempo, Prometheus and Loki are
  internal-only on purpose; Grafana is the only intended window into them.
- Docs: only 3 READMEs (root, `generators/`, `gateway-and-backends/`), short; no per-folder or per-dashboard `.md`. Recording rules
  (`config/prometheus-rules/`) are the semantic layer the stat/tile panels read; change semconv handling
  there, not in each panel.
- The label contract (service.namespace = application, host.name = server, …) is defined in
  `generators/README.md` and enforced by `config/prometheus.yml` (`otlp.promote_resource_attributes`) — change
  them together.
