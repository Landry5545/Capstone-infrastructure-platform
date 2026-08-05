# Day 10: Monitoring Integration for New Services

## Overview

Goal: confirm Wiki.js, Snipe-IT, and Uptime Kuma are covered by the existing Prometheus/Grafana/Loki monitoring stack (Project 5 and Project 9), and add active uptime monitoring via Uptime Kuma itself.

## Step 1: Container Metrics (cAdvisor)

No configuration changes needed. cAdvisor monitors at the Docker daemon level, so it automatically picked up `wikijs`, `wikijs-db`, `snipeit`, `snipeit-db`, and `uptime-kuma` alongside existing containers. Verified via the "cAdvisor Docker Insights" Grafana dashboard — all five new containers appear with clean restart histories.

## Step 2: Log Collection (Loki/Promtail)

Promtail's configuration lives in a separate project folder (`~/monitoring-stack/`, from Project 9) rather than the capstone repo. Its `docker` scrape job uses `docker_sd_configs` against `/var/run/docker.sock`, which — like cAdvisor — auto-discovers every running container. No config changes needed.

Verified via Grafana Explore (Loki data source, Code mode):
- `{container="wikijs"}` — logs present
- `{container="snipeit"}` — logs present (138 lines)
- `{container="uptime-kuma"}` — initially appeared empty on a 3-hour window; widening to 24 hours showed 281 lines. Not a bug — just sparse log activity in the shorter window.

## Step 3: Active Uptime Monitoring (Uptime Kuma)

Added three HTTP(s) monitors, checking every 60 seconds:
- **Wiki.js** → `http://wiki.lab.local/`
- **Snipe-IT** → `http://assets.lab.local/`
- **Uptime Kuma** → `http://status.lab.local/`

Monitors use the internal hostnames (no `:8081` port), since Uptime Kuma runs as a container on the same Docker network as Nginx and reaches it directly over port 80.

### Issue: DNS resolution failure inside uptime-kuma container

**Symptom:** Wiki.js monitor immediately failed with `getaddrinfo ENOTFOUND wiki.lab.local`.

**Root cause:** Same class of issue as the earlier client-vm2 DNS saga — the `uptime-kuma` container uses Docker's default DNS, which has no knowledge of the internal `lab.local` BIND9 zone.

**Fix:** Added an explicit `dns:` block to the `uptime-kuma` service in `docker-compose.yml`:
```yaml
  uptime-kuma:
    dns:
      - 10.10.10.10
      - 8.8.8.8
```
Recreated the container (`docker compose up -d --force-recreate uptime-kuma`). Wiki.js monitor went green on the next check cycle.

### Issue: Snipe-IT monitor timing out despite fast response

**Symptom:** Snipe-IT monitor showed "Down — timeout exceeded" even though direct `curl` tests from the host completed in under a second.

**Investigation:** Manually queried the endpoint from inside the `uptime-kuma` container using Node's built-in `http` module (container has no `curl`/`wget` binaries usable via `docker exec` in the expected way — actually had `curl` but not `wget`; used Node directly for a clean test). Result: `Status: 302, Location: http://assets.lab.local:8081/login`.

**Root cause:** Same `APP_URL` issue from the Day 7-9 Snipe-IT setup — the app is configured to redirect to `assets.lab.local:8081`, which is the Windows-host NAT port, meaningless from inside the Docker network. Uptime Kuma was following the redirect (Max Redirects was set to 10) and hanging trying to reach an unreachable address, eventually timing out.

**Fix:** Set **Max Redirects** to `0` on the Snipe-IT monitor, so the 302 itself (already within the accepted `300-399` status code range) counts as a successful check instead of being followed. Also reduced **Timeout** to 15s (down from the default) since real response time is ~400ms–3.5s. Monitor went green immediately, response time settled around 441ms.

## Outcome

All three services now have:
- Passive container metrics (cAdvisor → Prometheus → Grafana)
- Passive log aggregation (Promtail → Loki → Grafana)
- Active uptime/health checks (Uptime Kuma)

VirtualBox snapshot `day10-monitoring-integration-2026-08-05` taken on `server-vm` as a rollback point. This closes out the "wire new services into monitoring" step of the capstone roadmap.
