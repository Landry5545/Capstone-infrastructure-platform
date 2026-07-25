# Day 1 — Infrastructure Audit Findings

**Project:** Design, Consolidation, and Extension of an Automated IT Infrastructure Platform for a Small Organization
**Date:** 2026-07-25
**Scope:** Fresh-state audit of all 9 prior lab projects across `server-vm` (10.10.10.10) and `client-vm` (10.10.10.20), prior to adding FreeIPA and the three new internal services (Wiki.js, Snipe-IT, Uptime Kuma).

---

## 1. Method

Rather than relying on `docker ps` alone, the audit checked:
- Full container inventory on both VMs (`docker ps -a`, `docker compose ps`)
- Native (non-containerized) services via `systemctl status`
- Kernel logs for previously-reported hardware warnings (`dmesg`)
- Resource allocation (`nproc`, `free -h`)
- Known pending issues flagged from earlier projects

## 2. server-vm — Inventory & Status

| Service | Type | Status |
|---|---|---|
| cAdvisor | Docker container | Up, healthy |
| Prometheus | Docker container | Up |
| Grafana | Docker container | Up |
| BIND9 (`named.service`) | Native systemd | Active, listening on IPv4/IPv6 |
| GitHub Actions self-hosted runner | Native systemd | Active, connected to GitHub, listening for jobs |

No issues found on server-vm's container/service health.

## 3. client-vm — Inventory & Status

| Service | Type | Status |
|---|---|---|
| Promtail | Docker container | Up |
| `node_exporter` | Native systemd | Active, listening on `:9100` |
| ~~`test-nginx`~~ | Docker container | **Found stale (created 2026-07-05), removed** |

## 4. Findings & Resolutions

### 4.1 Stale test container on client-vm
**Finding:** `test-nginx` container present, not part of documented architecture, running since 2026-07-05 (~3 weeks, likely leftover from early Nginx testing in Project 2/3).
**Resolution:** Removed via `docker stop test-nginx && docker rm test-nginx`.
**Status:** ✅ Resolved

### 4.2 client-vm under-resourced (root cause of prior `soft lockup - CPU#0` warnings)
**Finding:** `client-vm` was allocated only 1 vCPU and 1.6GB RAM. A single-core VM under concurrent Docker + monitoring-agent load is a known trigger for kernel soft-lockup warnings. `sudo dmesg | grep -i "soft lockup"` returned no active recurrence at time of audit, but the resource constraint remained a standing risk ahead of adding FreeIPA-adjacent load.
**Resolution:** VM powered off; VirtualBox settings updated to 2 vCPUs / 4GB RAM. Confirmed post-boot via `nproc` (→ 2) and `free -h` (→ 3.6GB total).
**Status:** ✅ Resolved

### 4.3 Grafana admin password hardcoded in version-controlled file
**Finding:** `GF_SECURITY_ADMIN_PASSWORD=changeme123` was hardcoded in plaintext directly in `~/monitoring-stack/docker-compose.yml`, which is committed to the `monitoring-stack` GitHub repo.
**Resolution:**
- Created `.env` file (not committed) containing `GF_ADMIN_PASSWORD=changeme123`
- Updated `docker-compose.yml` to reference `${GF_ADMIN_PASSWORD}` instead of the literal value
- Added `.env` to `.gitignore`
- Recreated the Grafana container (`docker compose up -d grafana`) and verified the correct value resolves at runtime (`docker compose exec grafana printenv`)
**Status:** ✅ Resolved

### 4.4 Minor — obsolete `version` key in docker-compose.yml
**Finding:** Docker Compose emits a `WARN` that the top-level `version` attribute is obsolete and will be ignored.
**Resolution:** Not yet applied — flagged for cleanup, no functional impact.
**Status:** ⏳ Deferred (low priority)

## 5. Summary

| # | Issue | Severity | Status |
|---|---|---|---|
| 1 | Stale `test-nginx` container | Low | ✅ Resolved |
| 2 | client-vm under-resourced | Medium | ✅ Resolved |
| 3 | Grafana password hardcoded | Medium (security) | ✅ Resolved |
| 4 | Obsolete `version` key in compose file | Cosmetic | ⏳ Deferred |

**Day 1 outcome:** All 9 prior projects confirmed functional from a fresh-state check. Two of the two known pending issues (resource allocation, hardcoded credential) resolved during the audit rather than deferred to Day 2-3, since both required minimal additional effort once identified. Infrastructure is now in a clean, documented baseline state ahead of FreeIPA installation and new service deployment (Days 4+).
