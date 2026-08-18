# Day 11 Incident Report: Recurring Disk Corruption on client-vm2 During Docker Deploy

**Date:** 2026-08-10 to 2026-08-18
**Affected system:** client-vm2 (10.10.10.20), Ansible-managed deploy target
**Severity:** High — blocked capstone deploy for multiple days
**Status:** Resolved

## Summary

The Day 11 Ansible deploy (`capstone-services.yml`) requires pulling five Docker
images (postgres, wiki.js, mariadb, snipe-it, uptime-kuma) onto client-vm2 before
`docker-compose` can bring up the stack. What should have been a routine deploy
turned into a multi-day incident caused by a cascading chain of host disk
corruption, VirtualBox stability issues, and Docker/containerd metadata damage
that kept recurring even after apparently successful repairs.

## Timeline of failure modes

1. **Undersized disk.** client-vm2's original 8GB disk was too small for five
   Docker images plus growth headroom. A resize to 20GB was attempted
   in-place using `parted` + `pvresize` + `lvextend` + `resize2fs`.

2. **Host disk corruption surfaced during resize.** Repeated `ATA I/O errors`,
   GPT corruption, and `ext4_validate_block_bitmap` checksum failures occurred
   during the resize and subsequent Docker pulls. Root cause traced to the
   **Windows host's C: drive**, not the guest filesystem. `chkdsk C: /f /r`
   on the host resolved the underlying corruption.

3. **Docker/containerd metadata corruption (separate issue).** Independent of
   the filesystem corruption, `/var/lib/docker` and `/var/lib/containerd`
   became corrupted multiple times, producing `bbolt` database errors
   (`freelist`, `bucket` panics) and "blob not found" errors during pulls.
   Root cause: **abrupt VirtualBox window closures during active Docker
   writes** — VirtualBox would pause the VM (due to low host disk space or
   Windows sleep settings), and force-closing the window mid-write corrupted
   containerd's bbolt metadata store.

4. **Fix attempts that didn't hold.** Deleting and recreating
   `/var/lib/docker` and `/var/lib/containerd`, and running `fsck -y` on a
   still-mounted root filesystem, appeared to succeed but did not actually
   repair anything — `fsck` cannot safely repair a mounted root filesystem.
   The corruption pattern (`bad block bitmap checksum`, `EXT4-fs error`)
   returned on the next boot every time.

5. **Recovery via snapshot restore.** A pre-existing VirtualBox snapshot,
   `pre-ansible-deploy-2026-08-10`, taken before any of this began, provided
   a clean known-good state. Restoring it (with the VM fully powered off,
   and *not* preserving current state) gave a genuinely clean filesystem —
   confirmed via `sudo dmesg | grep -i ext4` returning no errors.

6. **Disk resize, done correctly this time.** The snapshot restore reverted
   the disk to its original 10GB size. The resize was redone, but this time
   targeting the **actual attached medium** — a differencing disk under
   `Snapshots\{uuid}.vdi`, not the base `client-vm.vdi` — discovered via
   `VBoxManage showvminfo client-vm2 --machinereadable`. Resizing the wrong
   file (the base disk) silently succeeds but has no effect on the guest,
   since the differencing disk chain still constrains the visible size.

7. **Soft lockup during image pull.** Mid-pull, the guest kernel logged
   `watchdog: BUG: soft lockup - CPU#0 stuck for 128s!`, freezing a
   `snipe/snipe-it` layer download for over two hours. Recovered by
   restarting the download after the lockup cleared; the image was re-pulled
   successfully on retry.

8. **Missing packages on rerun.** Once the images were cached, the Ansible
   playbook itself succeeded through Docker Compose, but failed twice more on
   simple missing-package issues: `nginx` and later `bind9`/`ufw` were not
   yet installed on client-vm2. Both were one-line `apt install` fixes,
   preceded by `sudo dpkg --configure -a` to clear an interrupted dpkg state.

## Resolution

* Host disk corruption fixed via `chkdsk C: /f /r` on Windows.
* Guest filesystem corruption fixed by restoring the `pre-ansible-deploy-2026-08-10`
  VirtualBox snapshot rather than continuing to patch a filesystem that kept
  re-corrupting.
* Disk correctly resized to 20GB by resizing the actual attached differencing
  disk, not the base VDI.
* All five Docker images pulled successfully after the resize.
* `nginx`, `bind9`, and `ufw` installed on client-vm2.
* `ansible-playbook -i inventory.ini capstone-services.yml` completed with
  `ok=12 changed=1 failed=0`.
* Verified all three services live via `curl -I`:
  * `wiki.lab.local` → `200 OK`
  * `assets.lab.local` → `302` → `/login` (Snipe-IT, session cookie issued)
  * `status.lab.local` → `302` → `/dashboard` (Uptime Kuma)

## Root causes

1. Host-level disk corruption (Windows C: drive) triggered by an undersized
   host disk under sustained I/O load.
2. VirtualBox auto-pausing the guest under host disk pressure, combined with
   force-closing the VM window during active Docker writes, repeatedly
   corrupted containerd's bbolt metadata store.
3. `fsck` was run against a mounted filesystem, giving false confidence that
   corruption was repaired when it was not.
4. VDI resize was initially applied to the wrong file in the base/differencing
   disk chain, silently succeeding without affecting the guest's visible disk
   size.

## Lessons learned / process changes

* **Never force-close a VirtualBox VM window during active disk I/O.** Always
  use a clean guest shutdown (`sudo shutdown now`) or let a paused VM resume
  before touching the window.
* **`fsck` must run against an unmounted (or read-only, via recovery mode)
  filesystem** to actually repair corruption. Running it against a live
  mounted root filesystem is a no-op that can mask real damage.
* **When corruption recurs after a fix, stop patching and restore from a
  known-good snapshot.** Repeated patch cycles on a genuinely damaged
  filesystem cost far more time than a clean restore.
* **For differencing/snapshot disk chains, always resize the currently
  attached medium**, found via `VBoxManage showvminfo --machinereadable`,
  not just the base VDI.
* **Keep host disk space comfortably above the threshold that triggers
  VirtualBox auto-pause** — low host disk space was an indirect cause of two
  separate failure modes in this incident.
* **Take a pre-deploy snapshot before any risky operation** (in this case,
  `pre-ansible-deploy-2026-08-10`) — it was the single thing that ultimately
  ended the incident.

## Time cost

Approximately 8 days elapsed from first corruption symptom to a clean,
verified deploy, spanning multiple failed repair attempts before the
snapshot-restore approach was adopted.
