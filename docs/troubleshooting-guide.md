# Troubleshooting Guide

This guide consolidates the real issues encountered while building this platform, grouped by category rather than by day. Each entry links back to the full incident write-up in `audit/` for complete command-by-command detail. If you're rebuilding this platform or hitting a similar symptom, start here.

## Recurring pattern: host resource starvation → real filesystem corruption

**This is the single most important lesson from this project.** It happened three separate times, on two different VMs, across two different days:

| When | VM | Trigger |
|---|---|---|
| Day 11 | client-vm2 | Disk resize + Docker image pulls under low host disk space |
| Day 12 (1st) | server-vm | Disk hit 100% full while registering a CI/CD runner |
| Day 12 (2nd) | server-vm | In-place VDI resize followed by live `resize2fs` |

**What actually happens:** when a VirtualBox guest goes without CPU time for too long — because the host is out of RAM, out of disk space, or the VM gets paused mid-operation — the guest kernel's soft-lockup watchdog fires (`watchdog: BUG: soft lockup - CPU#0 stuck for Ns!`). This is not cosmetic. Any disk write in flight when this happens can be corrupted at the block level. Symptoms seen: `ext4_validate_block_bitmap` checksum errors, `ext4_resize_begin: There are errors in the filesystem`, ATA I/O errors (`ata3.00: failed command: READ FPDMA QUEUED`, `SError: { DRDY ERR }`), and Docker's own `/var/lib/docker`/`/var/lib/containerd` metadata becoming corrupted independently of the filesystem.

**What does NOT reliably fix it:** running `fsck -fy` repeatedly on an already-corrupted filesystem. Multiple consecutive passes produced *different, non-converging* sets of "FILE SYSTEM WAS MODIFIED" results rather than settling to zero — each pass could introduce as much damage as it repaired if the VM paused again mid-check.

**What does fix it:**
1. Confirm host RAM headroom *before* starting any write-heavy operation (`Get-CimInstance Win32_OperatingSystem | Select FreePhysicalMemory` on the host).
2. If a VM pauses during a disk-write operation, **do not resume and continue** — power off and restart the operation from a known-good state (snapshot restore).
3. **Never resize a disk in place while it's attached.** Use `VBoxManage clonemedium` to produce a flat, standalone VDI first, resize the clone, then attach it. In-place resize of a differencing disk followed immediately by a live `resize2fs` was the specific combination that triggered pauses and corruption every time it was tried; the clone-then-resize approach completed cleanly with zero pauses across every subsequent attempt.
4. If corruption has already occurred, boot an Ubuntu Server live ISO, activate the LVM volume group (`vgscan && vgchange -ay`), and run `fsck -fy` against the *unmounted* logical volume — never against a live, mounted root filesystem.

Full detail: [`audit/day11-incident-disk-corruption-recovery.md`](../audit/day11-incident-disk-corruption-recovery.md), [`audit/day12-incident-disk-corruption-and-cicd-recovery.md`](../audit/day12-incident-disk-corruption-and-cicd-recovery.md)

## Host disk space

Running four VMs (`server-vm`, `client-vm2`, `client-vm` base disk, `freeipa-vm`) on a single Windows host drive is tight. A real deploy attempt (pulling 5 Docker images) failed twice purely because the host C: drive ran out of space mid-operation, auto-pausing the VM (`DrvVD_DISKFULL`).

- Check host free space before any multi-image Docker pull or VM disk resize.
- Deleting old VirtualBox snapshots reclaims the most space, but snapshot merges themselves need working room — a merge can fail with `E_OUTOFMEMORY` if there isn't enough temp space, which is a frustrating chicken-and-egg problem when you're deleting a snapshot specifically to free space.
- `docker system prune -af` frequently reclaims 0 bytes on a healthy running platform — if every container is actively `Up`, there's nothing unused to prune. Check `docker system df -v` before assuming pruning will help; it usually means the disk is genuinely undersized for the workload, not cluttered.

Full detail: [`audit/day11-disk-space-blocker.md`](../audit/day11-disk-space-blocker.md)

## VirtualBox VM pausing / sleep interactions

Two unrelated causes produce the same symptom (VM shows `[Paused]` in the VirtualBox title bar):

1. **Windows sleep settings.** If the host goes to sleep, every running VM pauses with it. Fix: Settings → Power & Sleep → "When plugged in, PC goes to sleep after" → **Never**, for any session involving VM work.
2. **Low host disk space**, which VirtualBox treats as a hard stop for any VM with an active write.

Either way: **always resume properly or run a clean `sudo shutdown now` from inside the guest.** Force-closing the VirtualBox window while Docker is mid-write is a confirmed, repeatable way to corrupt `/var/lib/docker` and `/var/lib/containerd` independently of any filesystem-level issue — fixed by stopping both services, moving both directories aside, and letting them rebuild clean.

## Clock desync after extended saved-state

A VM left in VirtualBox's saved state (or asleep on the host) for several hours comes back with its system clock still at the moment it was saved — not the current time. This is harmless to data (nothing was mid-write during the pause), but it breaks anything timestamp-sensitive:

- Kernel soft-lockup warnings on resume are usually just the guest catching up, not new damage — check `dmesg` for actual `ext4` errors to confirm before assuming the worst.
- `apt update` fails with `Release file ... is not valid yet` because the repository's signed timestamps are now "in the future" relative to the guest's stale clock.

Fix: `sudo timedatectl set-ntp off && sudo timedatectl set-ntp on` to force a resync, then retry.

## Playbook references a file that was never committed

Happened twice, to two different sets of files (`docker-compose.yml`/`.env` on Day 12's first pass, then `files/nginx/*.conf` and `files/bind9/db.lab.local` after a snapshot restore). The pattern: a file exists and works fine on the live server, the Ansible playbook references it under `files/`, but it was created directly on the VM (or via an editor session) and never actually `git add`ed to the `ansible-lab` repo. The playbook then fails with `Could not find or access 'files/...'` the first time it's run from a clean clone.

- Before trusting a playbook to be portable, clone the repo fresh somewhere and confirm every `src:` path under `files/` actually resolves — don't assume a working deploy means the repo is complete.
- Secrets belong in `.gitignore` (`files/.env`) and should be reconstructed at deploy time from GitHub Secrets, not committed — but *non-secret* config (Nginx site configs, DNS zone files) should be committed like any other source file.

## GitHub Actions workflow syntax

A single stray character — `${[ secrets.WIKIJS_DB_PASSWORD }}` instead of `${{ secrets.WIKIJS_DB_PASSWORD }}` — caused the `.env`-writing step to fail with a bash "bad substitution" error, while the four other secret lines above it (correctly formed) worked fine and were masked in the log as `***`. If one secret in a block fails while sibling lines succeed, check for a literal typo in the `${{ }}` expression syntax before suspecting the secret itself is missing or misnamed.

## Self-hosted runner registration

- Registration tokens from GitHub's "New self-hosted runner" page expire in about an hour. If `config.sh` fails with a 404 against GitHub's API, the token has expired — grab a fresh one rather than retrying the same command.
- If a runner's local `.runner`/`.credentials` files already exist from a prior attempt, `config.sh` will detect "a runner exists with the same name" and prompt to replace it — safe to accept if you know the prior attempt didn't complete cleanly.
- A runner that shows **green "Idle"** immediately after `svc.sh start` can still be dead moments later if the registration was invalidated server-side (e.g. GitHub cleaned up a stale/duplicate entry) — check `sudo systemctl status` a few minutes later, not just right after starting, and cross-check the Runners page in GitHub's UI for "Offline" before trusting a local "active (running)" status alone.

## Terminal-only VM has no way to test a web UI

`client-vm2` has no GUI browser installed. Rather than installing one, add the internal DNS names to the **Windows host's** hosts file (`C:\Windows\System32\drivers\etc\hosts`) pointing at `server-vm`'s IP, then browse from the host's own Firefox/Edge. Works for one-time setup wizards (Wiki.js, Snipe-IT, Uptime Kuma admin account creation) without needing X11 forwarding or a desktop environment on the VM.

## DNS resolution silently falling back to public DNS

`client-vm2` intermittently failed to resolve `*.lab.local` names despite `systemd-resolved` appearing correctly configured. Root cause: `/etc/netplan/50-cloud-init.yaml` had a hardcoded `nameservers: addresses: [8.8.8.8, 1.1.1.1]` — a config generated at VM creation time that silently overrode the intended DNS server on every netplan apply, including after reboots. `systemd-resolved` status output looked fine but was still ultimately obeying the stale netplan config.

Fix used: directly overwrite `/etc/resolv.conf` with `nameserver 10.10.10.10` then `nameserver 8.8.8.8` as a fallback. Caveat carried forward: `/etc/resolv.conf` is normally a symlink managed by `systemd-resolved`, so this fix should be re-verified after any reboot rather than assumed permanent — the netplan file is the actual root cause and is the more durable place to fix it if this recurs.

## "Working" service that turns out to have a missing dependency

Nginx reverse proxy configs existed and were referenced throughout the project, but Nginx itself wasn't actually installed on `server-vm` when Day 7-9 service deployment began — despite an earlier project (`nginx-reverse-proxy`) supposedly having set it up. `sudo apt install nginx -y` also needed `sudo dpkg --configure -a` first to clear an unrelated interrupted dpkg state. Lesson: a component being "in scope from an earlier project" doesn't mean it survived on the actual running VM — verify with `systemctl status`/`which`, don't assume.

## FreeIPA: OS choice matters more than expected

FreeIPA has no native package on Ubuntu (the community PPA is abandoned and unsupported on any current release), and running it in Docker on Ubuntu hits a cascade of systemd-in-container problems (cgroup v1/v2 mismatch, read-only cgroup mounts, unresolvable D-Bus errors) that aren't really fixable without fighting the container runtime itself. Installing on **AlmaLinux 9** — FreeIPA's actual native/supported platform — avoided all of it; `dnf install freeipa-server` and `ipa-server-install` worked without drama. If a tool has one specific "native" OS, don't spend days forcing it onto a different one for consistency's sake.

Full detail: [`audit/freeipa-installation-saga.md`](../audit/freeipa-installation-saga.md)

## Quick reference: what to check first

| Symptom | Check this first |
|---|---|
| VM shows `[Paused]` | Host sleep settings, then host free disk space |
| `ext4` errors in `dmesg` | Was the VM paused or under RAM/disk pressure recently? |
| `resize2fs` fails after `growpart`/`pvresize`/`lvextend` succeed | `dmesg \| grep ext4` — likely pre-existing corruption, not a resize bug |
| Ansible fails with `Could not find or access 'files/...'` | `git log --all -- "files/..."` — was it ever actually committed? |
| One secret in a `.env`-writing step fails, others succeed | Check the `${{ }}` syntax on that specific line for a typo |
| `apt update` says a Release file "is not valid yet" | Guest clock drift — run `timedatectl status`, resync NTP |
| Runner shows "Idle" but jobs never run | Recheck GitHub's Runners page for "Offline" — local service status can lag |
| DNS resolves inconsistently on a VM | Check netplan's `nameservers:` block, not just `systemd-resolved` status |
