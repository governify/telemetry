# Telemetry stack — working rules

Hand-maintained observability stack (Caddy + OTel Collector + Tempo + Prometheus + Loki + Grafana)
fronting `telemetry.governify.io`. Keep `landing/index.html` and `README.md` in sync — never let
them go stale. No boilerplate.

## Sync rule

Any change to `telemetry-docker-compose.yaml` or `config/Caddyfile` that adds, removes, exposes,
or un-exposes a service **must** update both files in the same change, not as a follow-up:

- New Caddy subdomain → add an entry to `landing/index.html` (same style as the existing entries,
  `target="_blank" rel="noopener"`).
- Service made internal-only → remove its landing page entry or mark it `class="disabled"` (see
  the Tempo/Prometheus/Loki entries for the pattern) — never leave a dead link.
- New/removed compose service → update the README's service list.

Don't ask before doing this — it's expected as part of finishing the change.

## Style notes (from prior feedback in this project)

- Landing page stays minimal: title + a plain link list. No footer, no status badges, no
  subtitle — explicitly rejected before as looking AI-generated.
- After editing a bind-mounted config file, prefer `up -d --force-recreate <service>` over
  `reload`/`restart` alone — single-file bind mounts here go stale because the editor replaces
  the file (new inode) rather than writing in place. Verify with
  `diff <(docker exec <container> cat <path>) <local path>`.
- Don't expose a service externally unless explicitly asked — Tempo, Prometheus and Loki are
  internal-only on purpose; Grafana is the only intended window into them.
