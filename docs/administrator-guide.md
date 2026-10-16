# Administrator Guide

Day-to-day operational reference for running this platform after it's installed. For first-time setup, see [`installation-guide.md`](./installation-guide.md). For diagnosing a specific failure, see [`troubleshooting-guide.md`](./troubleshooting-guide.md).

## Accessing the Platform

| Service | URL | Notes |
|---|---|---|
| Wiki.js | http://wiki.lab.local | Internal documentation |
| Snipe-IT | http://assets.lab.local | IT asset management |
| Uptime Kuma | http://status.lab.local | Service health dashboard |
| Grafana | http://server.lab.local:3000 (or via its own container port) | Metrics dashboards |
| FreeIPA web UI | https://freeipa-vm.lab.local | Identity management |

All `*.lab.local` names only resolve on machines using server-vm (10.10.10.10) as their DNS server, or on the Windows host if its hosts file has been edited manually.

## Routine Health Checks

**Containers (server-vm):**
```bash
docker compose ps
```
All five application containers, plus the monitoring stack containers, should show `Up`. A container cycling through restarts is the first sign of a resource or config problem — check `docker compose logs <service>` next.

**Disk space (server-vm and client-vm2):**
```bash
df -h /
```
This platform has a documented history of disk-full conditions leading to real filesystem corruption (see the troubleshooting guide). Treat anything above 80% used as worth investigating immediately, not deferring.

**Memory:**
```bash
free -h
```
server-vm runs 11+ containers; under 500MB available free RAM is a warning sign, not just a number to note.

**Filesystem integrity:**
```bash
sudo dmesg -T | grep -i ext4
```
Should return nothing beyond the normal boot-time orphan-cleanup and remount lines. Any `ext4_validate_block_bitmap` or similar error means stop what you're doing and consult the troubleshooting guide's corruption-recovery procedure before continuing.

**CI/CD runner status:**
```bash
sudo systemctl status actions.runner.<org>-ansible-lab.<runner-name>.service
```
Also check the Runners page in the repo's GitHub Settings — a runner can show "active (running)" locally while GitHub has independently marked it offline; the GitHub UI is the source of truth.

## Managing FreeIPA Users

**Add a user:**
```bash
ipa user-add <username> --first=<First> --last=<Last> --email=<email>
```

**Add a group and assign a user:**
```bash
ipa group-add <groupname>
ipa group-add-member <groupname> --users=<username>
```

**Reset a user's password:**
```bash
ipa passwd <username>
```

**Verify a user resolves correctly:**
```bash
id <username>
```

**Remove a user:**
```bash
ipa user-del <username>
```

All of the above require an active admin Kerberos ticket — run `kinit admin` first if `klist` shows no valid ticket.

## Triggering a Redeploy

**Automatic (preferred):** push any change touching `capstone-services.yml` or `inventory.ini` in the `ansible-lab` repository to `main`. The self-hosted runner picks it up automatically; check progress in the repo's Actions tab.

**Manual (if the runner is down or for a one-off change):**
```bash
cd ~/ansible-lab
ansible-playbook -i inventory.ini capstone-services.yml --check   # dry run first
ansible-playbook -i inventory.ini capstone-services.yml           # then real
```

## Adding or Rotating a Secret

1. Update the value in the repository's GitHub Settings → Secrets and variables → Actions.
2. If it's a new secret (not a rotation), also add it to the `cat > files/.env << EOF ... EOF` block in `.github/workflows/deploy-capstone.yml` so it actually gets written at deploy time.
3. Trigger a redeploy to apply it. Secrets never need to be touched directly on the VM — they're reconstructed fresh from GitHub on every deploy.

## Backup and Recovery

**What to back up, and how often:**

| Item | Method | Suggested frequency |
|---|---|---|
| Wiki.js / Snipe-IT databases | `docker exec <container> pg_dump` / `mysqldump` to a file outside the container | Weekly, or before any major change |
| `docker-compose.yml`, `.env`, Nginx configs, BIND9 zone | Already version-controlled in `ansible-lab` — confirm `git status` is clean | Continuous (git is the backup) |
| FreeIPA directory data | `ipa-backup` (built-in FreeIPA tool) | Before any FreeIPA upgrade or config change |
| VM-level safety net | VirtualBox snapshot, taken before any disk resize or risky operation | Before any disk-level change — see troubleshooting guide, this was the single most important safeguard during development |

**Restoring a database:**
```bash
docker exec -i <container> psql -U <user> -d <db> < backup.sql        # Postgres (Wiki.js)
docker exec -i <container> mysql -u <user> -p<pass> <db> < backup.sql # MariaDB (Snipe-IT)
```

**Restoring from a VirtualBox snapshot (host side):**
```powershell
VBoxManage snapshot <vm-name> restore <snapshot-name>
```
Confirm the VM's state is `poweroff` first (`VBoxManage showvminfo <vm> --machinereadable | Select-String VMState`), or the restore will fail.

> **Note:** backup and restore procedures above are documented and ready to use, but have not yet been exercised as a full drill (restoring a database from a cold backup, or restoring a VM from a snapshot taken specifically for this purpose). Running that drill once, end to end, is recommended before relying on this platform for anything beyond lab/portfolio use.

## Verifying Survival of a Reboot

Services are started by Docker Compose, which is not configured to launch automatically on boot by default. After any planned reboot of server-vm or client-vm2:

```bash
sudo systemctl status docker        # should be active
cd ~/capstone-infrastructure-platform && docker compose ps
```

If containers did not come back up automatically, bring them up manually:
```bash
docker compose up -d
```

To make this automatic going forward, enable Docker's own restart policy on each service in `docker-compose.yml` (`restart: unless-stopped`) rather than relying on a manual check after every reboot.

## Common Maintenance Tasks

**Renew/check UFW rules:**
```bash
sudo ufw status numbered
```

**Check which hostnames BIND9 currently serves:**
```bash
cat /etc/bind/db.lab.local
```
Bump the zone file's serial number after any edit, then `sudo systemctl reload bind9`.

**Free up disk space safely:**
```bash
sudo journalctl --vacuum-size=200M
docker system df -v        # check what's actually reclaimable before pruning
docker system prune -af    # only if the above shows genuinely unused images/containers
```
On a healthy, fully-running platform, `prune` often reclaims 0 bytes — that's expected, not a sign something is broken, and does not mean the disk isn't genuinely low.

**Check Grafana dashboard health:** open Grafana, confirm the node_exporter and cAdvisor data sources show recent data points for both server-vm and client-vm2, not just server-vm.
