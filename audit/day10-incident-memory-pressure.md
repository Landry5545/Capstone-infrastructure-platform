# Incident: Snipe-IT Intermittent 502s Due to server-vm Memory Exhaustion

## Summary

Shortly after Day 10 monitoring integration was completed, Uptime Kuma flagged Snipe-IT as intermittently down with `502 Bad Gateway` and `request failed with status code 502`. The issue was traced to `server-vm` running low on memory after accumulating 11 containers (5 application services + the full Prometheus/Grafana/Loki/cAdvisor/node_exporter monitoring stack) on a VM originally sized for a smaller footprint.

## Symptoms

- Uptime Kuma: Snipe-IT monitor flapping between Up/Down, eventually settling on "Down — Request failed with status code 502"
- Browser: consistent `502 Bad Gateway` from Nginx when loading `assets.lab.local`
- Confusingly, manual `curl -I -H "Host: assets.lab.local" http://localhost` from the host **succeeded** with `200 OK` multiple times during the outage window

## Investigation

1. `docker ps -a --filter "name=snipeit"` — container showed "Up" the whole time, not crash-looping
2. `docker logs snipeit` — no errors, just routine Laravel scheduler output (`No scheduled commands are ready to run`, successful `auth:clear-resets` cron run)
3. `tail /var/log/nginx/error.log` — repeated errors over a ~17 minute window:

recv() failed (104: Connection reset by peer) while reading response header from upstream, upstream: "http://127.0.0.1:3002/"
upstream prematurely closed connection while reading response header from upstream

   This pattern — Nginx successfully connecting to the upstream but the connection being reset mid-response — pointed away from a networking/DNS/config issue (already ruled out during initial Day 7-9 setup) and toward the upstream process itself being killed or stalling under resource pressure.
4. `docker stats --no-stream` and `free -h` confirmed the root cause:

Mem: 1.6Gi total, 806Mi used, 74Mi free, 589Mi swap used

   `server-vm` was allocated only ~1.6GB RAM, with just 74MB genuinely free and the system already relying on swap. Under this pressure, PHP-FPM workers for Snipe-IT (and potentially other services) were intermittently reset — explaining the "works sometimes, fails other times" pattern that a simple "is it up or down" check couldn't fully capture on its own.

## Fix

Increased `server-vm`'s VirtualBox memory allocation:
1. Shut down `server-vm` cleanly (`sudo shutdown now`)
2. VirtualBox Manager → `server-vm` → Settings → System → Motherboard → increased Base Memory to 4096 MB (from ~1.6-2GB)
3. Confirmed at least 2 vCPUs allocated under the Processor tab
4. Restarted the VM

Post-fix, `free -h` showed substantially more headroom, and Snipe-IT stabilized to a consistent "Up" status in Uptime Kuma.

## Root Cause

`server-vm` was never resized after incrementally accumulating 9 prior lab projects' worth of services plus the capstone's 3 new application containers and FreeIPA-adjacent load. The VM's original memory allocation was sized for an earlier, lighter workload and was never revisited as the platform grew — a classic capacity-planning gap. This mirrors a similar issue flagged for `client-vm` during the Day 1 audit (unresolved soft lockup warnings, noted as needing a vCPU/RAM bump before adding more load) — the same class of problem, just surfacing on the other VM once it took on the new services.

## Lessons Learned

- **Uptime Kuma's status-code check alone wasn't enough to diagnose this** — it correctly flagged the outage but the real signal was in Nginx's error log (connection resets, not timeouts or DNS failures) combined with host-level resource metrics. A "service is down" alert is a starting point, not the full diagnosis.
- **`docker stats` and `free -h` should be a standard first check** for any "intermittent 502/504" symptom before chasing application-level config, since the pattern (works under manual single-request testing, fails under monitor's repeated polling) is a strong tell for resource exhaustion rather than a config bug.
- **VM sizing should be revisited at major consolidation milestones**, not just at initial provisioning — this platform grew from 2 lightly-loaded lab VMs to a single VM running 11 containers without a corresponding resource review until this incident forced it.

## Outcome

`server-vm` RAM increased from ~1.6GB to 4GB. Snipe-IT, and by extension the rest of the stack sharing that host, now has adequate headroom. No further 502s observed after the fix.
