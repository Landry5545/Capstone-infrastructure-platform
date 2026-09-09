# Day 12 Incident Report: server-vm Disk Corruption and CI/CD Recovery

**Date:** September 4–9, 2026
**Affected system:** server-vm (Ansible control node, 10.10.10.10)
**Related prior incident:** [day11-incident-disk-corruption-recovery.md](./day11-incident-disk-corruption-recovery.md) (client-vm2, August 2026)

## Summary

While extending the Day 11 GitHub Actions runner to redeploy capstone services, server-vm's root disk filled to 100%, triggering kernel soft lockups and genuine ext4 filesystem corruption — the same failure pattern already documented on client-vm2 during Day 11. Recovery required two separate corruption events, several failed in-place disk resize attempts, a full snapshot restoration, and the discovery that several files referenced by the Ansible playbook (`files/nginx/*.conf`, `files/bind9/db.lab.local`) had never been committed to the `ansible-lab` repository. Once the disk was stable and the repo complete, the GitHub Actions runner was re-registered and the CI/CD pipeline was verified working end-to-end with a successful automated deploy.

## Timeline

### Initial corruption (root cause: disk-full → kernel soft lockup → ext4 corruption)
- server-vm's 18GB disk filled to 100% while registering a second GitHub Actions runner and preparing `.env` reconstruction for the `ansible-lab` CI/CD workflow.
- The out-of-space condition triggered severe kernel soft lockups and RCU stalls on both CPUs.
- `dmesg` confirmed genuine ext4 corruption: `ext4_validate_block_bitmap` errors, bad block bitmap checksums, and `ext4_resize_begin: There are errors in the filesystem, so online resizing is not allowed`.
- This is the second occurrence of the same failure chain first seen on client-vm2 in Day 11: host RAM/CPU starvation → kernel soft lockup → ext4 corruption on the guest.

### Failed in-place resize attempts
- Resized server-vm's disk from 18GB to 40GB via `VBoxManage modifymedium --resize` on the attached differencing disk.
- `growpart`/`pvresize`/`lvextend` succeeded, but `resize2fs` failed against the already-corrupted filesystem.
- Repeated live-ISO `fsck -fy` passes (booting an Ubuntu Server live environment, activating the LVM volume group, running e2fsck directly on the unmounted logical volume) produced inconsistent, non-converging results — each pass reported different sets of "FILE SYSTEM WAS MODIFIED" block bitmap differences rather than settling to zero.
- The VM repeatedly **paused** mid-operation during `resize2fs` and during back-to-back `fsck` runs, even with the host showing several GB of free RAM and the VM allocated 3072MB. Each pause coincided with fresh ATA/SATA I/O errors (`ata3.00: failed command: READ FPDMA QUEUED`, `SError: { DRDY ERR }`) — evidence that VirtualBox's virtual storage controller was dropping in-flight I/O during the pause, not just a cosmetic freeze.
- Conclusion: an **in-place VDI resize followed immediately by a live `resize2fs`** is unreliable in this VirtualBox environment under any host-pause condition, and repeated fsck cycling on an already-inconsistent filesystem does not converge — it can make things worse.

### Recovery: snapshot restore + clone-based resize
- Restored the last known-good VirtualBox snapshot, `day10-monitoring-integration-2026-08-05`, returning server-vm to its original 18GB disk with all core services (Wiki.js, Snipe-IT, Uptime Kuma, monitoring stack) intact and verified clean (`dmesg` showed no ext4 errors; only normal boot-time orphan cleanup and remount messages).
- Investigated whether the resulting disk pressure was avoidable: `docker system df -v` showed 0% reclaimable space across images, containers, and volumes — the ~16GB in use was legitimate, active data backing 11 running containers, not bloat. The 18GB disk was genuinely undersized for this stack.
- Resized the disk safely this time using a different method: **`VBoxManage clonemedium`** to merge the differencing disk into a new flat, standalone 20GB VDI, then `VBoxManage modifymedium --resize 40960` on that standalone clone *before* attaching it to the VM — rather than resizing the differencing disk in place.
- This avoided the failure pattern entirely: the clone operation performed the bulk sequential copy as one safe, uninterrupted step, so the subsequent `growpart`/`pvresize`/`lvextend`/`resize2fs` chain inside the guest completed cleanly with no pause and no I/O errors.
- Verified with a full reboot (to exercise `ext4lazyinit` against the newly extended blocks) and a `dmesg` check — clean.

### Missing repository files
- Once server-vm was stable, re-running the Ansible playbook from client-vm2 failed repeatedly on missing source files: `files/nginx/wiki.lab.local.conf`, `files/nginx/assets.lab.local.conf`, `files/nginx/status.lab.local.conf`, and `files/bind9/db.lab.local`.
- `git log --all -- "files/nginx/*"` confirmed these files had **never been committed** to the `ansible-lab` repository — they existed only as live configuration on server-vm's filesystem, the same pattern already identified for `docker-compose.yml` and `.env` during the original Day 12 work.
- Recovered each file's content directly from server-vm's live configuration (`/etc/nginx/sites-available/*`, `/etc/bind/db.lab.local`), recreated them under `files/nginx/` and `files/bind9/` in the repo, and committed them.
- The playbook then completed a full clean run (`ok=13 changed=3 failed=0`), and `curl` checks against `wiki.lab.local`, `assets.lab.local`, and `status.lab.local` confirmed all three services responding correctly through Nginx.

### GitHub Actions runner re-registration and CI/CD verification
- The Day 11 snapshot restore also removed the previously-registered `actions-runner-ansible-lab` service, since that setup postdated the restored snapshot.
- Re-registered a fresh self-hosted runner (`server-vm-ansible-lab-new` — an earlier registration attempt failed with an expired token and left a stale, offline entry on GitHub that had to be superseded).
- Installed as a systemd service (`sudo ./svc.sh install && sudo ./svc.sh start`) so it persists across reboots and disconnects.
- Test push revealed one remaining bug in `deploy-capstone.yml`: the `.env`-writing step referenced `${[ secrets.WIKIJS_DB_PASSWORD }}` — a stray `[` in place of the second `{`, causing a bash "bad substitution" error. Corrected to `${{ secrets.WIKIJS_DB_PASSWORD }}`.
- Final push (commit `86d8383`, "Retest CI/CD after fixing .env secret syntax") triggered the workflow, which completed successfully end-to-end: checkout → `.env` reconstructed from the 5 GitHub Secrets → `ansible-playbook -i inventory.ini capstone-services.yml` run via the self-hosted runner → deploy succeeded.

### Unrelated finding: clock desync after VM save-state
- Separately, both server-vm and client-vm2 exhibited kernel soft-lockup warnings and `apt update` failures ("Release file ... not valid yet") after being left in VirtualBox's saved state for an extended period.
- Root cause: the guest's system clock had drifted roughly 2 days 9 hours behind real time across the save/resume gap, which invalidated APT's repository signature timestamps.
- Resolved with `sudo timedatectl set-ntp off && sudo timedatectl set-ntp on` to force a resync. Confirmed harmless to filesystem integrity (no disk I/O in flight during the saved-state period, no ext4 errors on either VM afterward).

## Root Cause

Two independent factors, now confirmed to recur across two separate VMs:

1. **Host resource starvation causes real filesystem corruption, not just a hang.** Whenever a VM's guest OS goes for an extended period without CPU time (from host RAM/CPU contention, or a long VirtualBox pause), the kernel's soft-lockup watchdog fires, and any in-flight disk write can be corrupted — this is not cosmetic. It has now caused genuine ext4 corruption on client-vm2 (Day 11) and server-vm (Day 12) under the same mechanism.
2. **In-place VDI resize + live `resize2fs` is an unsafe operation pattern in this environment.** Growing a differencing disk in place and then immediately running `resize2fs` on the live, mounted filesystem is disproportionately likely to trigger a host pause during the write-heavy resize step, with a real risk of I/O-level corruption if that happens. Cloning to a flat, standalone disk first — performing the resize on the clone before it's ever attached and mounted — avoided the issue entirely in this environment.

## Corrective Actions and Recommendations

- **Never resize a disk in place while it's attached to a running or soon-to-run VM.** Use `VBoxManage clonemedium` to produce a flat standalone VDI, resize the clone, then attach it — this was the only resize approach that completed reliably in this environment.
- **Treat a VM pause during any disk-write operation (resize2fs, fsck, large Docker pulls) as a hard stop, not something to resume through.** Power off and restart the operation from a known-good state rather than continuing past a pause.
- **Confirm host RAM headroom before any write-heavy operation.** Close unnecessary host applications and check `Get-CimInstance Win32_OperatingSystem | Select FreePhysicalMemory` before starting a resize, a large `apt`/Docker pull, or an Ansible deploy.
- **Commit every file the Ansible playbook references, not just the playbook itself.** Two consecutive incidents (Day 12's `.env`/`docker-compose.yml`, and this incident's `files/nginx/*` and `files/bind9/db.lab.local`) were caused by configuration files that existed only locally and were never pushed to git. A pre-flight check (`git status` for untracked files under `files/`, or a CI step that fails fast on missing `src:` targets) would catch this before a deploy attempt rather than during one.
- **Use `savestate` or a clean `poweroff` before leaving VMs idle for hours**, and run a manual NTP resync (`timedatectl set-ntp off && ... set-ntp on`) after resuming from an extended saved state, before running `apt update` or anything security-timestamp-sensitive.
- **Provision server-vm's disk generously from the start.** `docker system df` confirmed the original 18GB was undersized for the actual service footprint (11+ containers, full monitoring stack) with 0% reclaimable — this was not a cleanup problem, it was a sizing problem, and should be sized at deploy time rather than discovered under pressure.

## Verification

- `df -h /` on server-vm: 38G total, 16G used, 21G available, 44% used — stable across multiple reboots.
- `dmesg -T | grep -i ext4`: clean, no errors, on every boot since the clone-based resize.
- `curl -I` against `wiki.lab.local`, `assets.lab.local`, `status.lab.local`: all returning expected HTTP responses (200, 302, 302) through Nginx.
- GitHub Actions workflow run `86d8383` ("Retest CI/CD after fixing .env secret syntax"): green, completed successfully via the self-hosted runner, full Ansible deploy verified end-to-end.
