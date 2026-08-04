# Service Deployment Saga: Wiki.js, Snipe-IT, Uptime Kuma

## Overview

**Goal:** Deploy three internal services on `server-vm` (10.10.10.10) as part of the capstone consolidation — Wiki.js (documentation), Snipe-IT (asset management), and Uptime Kuma (uptime monitoring) — reverse-proxied through Nginx with clean `.lab.local` hostnames.

**Services deployed:**
| Service | Container Port | Nginx Hostname |
|---|---|---|
| Wiki.js | 3001 (Postgres-backed) | wiki.lab.local |
| Snipe-IT | 3002 (MariaDB-backed) | assets.lab.local |
| Uptime Kuma | 3003 (self-contained) | status.lab.local |

All five containers (including the two databases) were brought up via a single `docker-compose.yml` at the repo root. What followed was a chain of smaller infrastructure gaps, each blocking the next step until resolved.

---

## Issue 1: DNS Resolution Failures on client-vm2

**Symptom:** `client-vm2` could not resolve `wiki.lab.local`, `assets.lab.local`, or `status.lab.local`, despite BIND9 on `server-vm` having correct A records (serial bumped to 4) and `systemd-resolved` appearing correctly configured with the `~lab.local` routing domain.

**Root cause:** `/etc/netplan/50-cloud-init.yaml` on `client-vm2` had `nameservers.addresses` hardcoded to `[8.8.8.8, 1.1.1.1]` — public DNS with no awareness of the internal `lab.local` zone. `systemd-resolved` was technically configured correctly at the resolved level, but the underlying netplan config it inherited from never pointed at BIND9 (10.10.10.10).

**Fix:** Bypassed `systemd-resolved` entirely for this troubleshooting pass. Directly wrote `/etc/resolv.conf`:

nameserver 10.10.10.10
nameserver 8.8.8.8


**Caveat:** `/etc/resolv.conf` is normally a symlink managed by `systemd-resolved` and may be overwritten on reboot. This should be rechecked after any reboot of `client-vm2`, and ideally fixed properly at the netplan level in a future pass.

---

## Issue 2: Nginx Not Actually Installed on server-vm

**Symptom:** Reverse proxy setup couldn't begin — Nginx wasn't running, despite Project 3 (`nginx-reverse-proxy`) supposedly having installed and configured it.

**Root cause:** Unclear — Nginx was simply absent from `server-vm`. Possibly removed or never persisted across some earlier environment change.

**Fix:**
1. `sudo dpkg --configure -a` — needed first to clear an interrupted dpkg state that blocked the install
2. `sudo apt install nginx -y`
3. Created reverse proxy configs in `/etc/nginx/sites-available/` for `wiki.lab.local`, `assets.lab.local`, `status.lab.local` (Uptime Kuma's config includes WebSocket headers for its live dashboard updates)
4. Symlinked all three into `/etc/nginx/sites-enabled/`
5. `sudo nginx -t` confirmed config validity

---

## Issue 3: UFW Blocking Port 80

**Symptom:** Even after Nginx was installed and configured correctly, none of the three sites were reachable from `client-vm2`.

**Root cause:** UFW only allowed ports 22, 53, and 9100 — port 80 (HTTP) had never been opened.

**Fix:**

sudo ufw allow 80/tcp


This was the final blocker for internal (VM-to-VM) access. Verified end-to-end from `client-vm2`: `wiki.lab.local` returned HTTP 200, `assets.lab.local` and `status.lab.local` returned 302 redirects to their respective setup/dashboard pages — all expected behavior for first-run apps.

---

## Issue 4: Browser-Based Setup Wizards Inaccessible from Windows Host

**Symptom:** `client-vm2` has no GUI browser installed (terminal-only), so the Wiki.js/Snipe-IT/Uptime Kuma first-run setup wizards — which require a real browser — couldn't be completed from inside the lab network directly.

**Root cause:** `labnet` is a VirtualBox **internal network**, isolated from the Windows host by design. The host has no route to `10.10.10.10`, so even after adding a hosts file entry pointing `*.lab.local` at that IP, Windows Firefox couldn't connect (`connection timed out`).

**Fix:** Used NAT port-forwarding on `server-vm`'s NAT adapter (same technique already in use for Grafana on port 3000):
- Host Port `8081` → Guest Port `80`

**Sub-issue:** Initially attempted to reuse host port `8080`, which caused a "port forwarding rules not valid" conflict. Investigation via `sudo ss -tulnp | grep 8080` and `sudo docker ps` revealed port 8080 was already bound by **cAdvisor** (from the Project 5 monitoring stack) — not a stale/orphaned rule as first suspected. Correctly left that rule alone and used `8081` for the new HTTP forwarding instead.

Updated Windows hosts file (`C:\Windows\System32\drivers\etc\hosts`):

127.0.0.1 wiki.lab.local assets.lab.local status.lab.local


Browsing to `http://wiki.lab.local:8081`, `http://assets.lab.local:8081`, and `http://status.lab.local:8081` from the Windows host successfully reached all three setup wizards.

---

## Issue 5: Snipe-IT APP_URL Hardcoded Without Port

**Symptom:** During the Snipe-IT setup wizard, every step transition (`/setup` → `/setup/migrate` → `/setup/database`) silently dropped the `:8081` port from the URL and switched to `https`, resulting in "Unable to connect" errors partway through setup.

**Root cause:** `docker-compose.yml` had `APP_URL` hardcoded directly in the `snipeit` service's `environment:` block as `https://assets.lab.local` — no port, and `https` despite no SSL/TLS being configured anywhere in the stack (SSL/TLS is explicitly out of scope for this capstone). Every internal redirect Snipe-IT generated used this stale value instead of the actual access URL.

**Fix:**
1. Edited `docker-compose.yml`, changed the line to:

APP_URL: http://assets.lab.local:8081

2. Recreated the container to apply the change:

sudo docker compose up -d --force-recreate snipeit

3. Cleared Laravel's cached config, since the app had already cached the old value on first boot:

sudo docker exec snipeit php artisan config:clear
sudo docker exec snipeit php artisan cache:clear


**Note:** The Configuration Check page continued to display a stale "your real URL is http://..." warning even after the fix — this turned out to be non-blocking. The database tables had already been created from an earlier partial setup attempt, so "Nothing to migrate" appeared on Step 2, and setup completed successfully from Step 3 (Create Admin User) onward.

---

## Outcome

All three services fully deployed with admin accounts created and verified working:
- **Wiki.js** — reachable at `http://wiki.lab.local:8081`, admin account created
- **Snipe-IT** — reachable at `http://assets.lab.local:8081`, dashboard active
- **Uptime Kuma** — reachable at `http://status.lab.local:8081`, ready for monitor configuration

VirtualBox snapshot `services-deployed-2026-08-04` taken on `server-vm` as a rollback point before proceeding to Day 10 (wiring these services into the existing Prometheus/Grafana/Loki monitoring stack).
