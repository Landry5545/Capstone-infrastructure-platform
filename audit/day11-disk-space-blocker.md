# Day 11 Validation Blocked: Host and Guest Disk Space Exhaustion

## Summary

While attempting the real (non-`--check`) run of `capstone-services.yml` against `client-vm2` — the final validation step for Day 11 — the VM was automatically paused by VirtualBox with a `DrvVD_DISKFULL` error. Investigation traced this to the Windows host's C: drive being completely full (0 bytes free out of 237GB), which prevented `client-vm2`'s differencing disk from growing to accommodate the Docker image pulls (Wiki.js, Snipe-IT, MariaDB, Uptime Kuma) triggered by the playbook's `docker compose up -d` task.

## Timeline

1. Ran `ansible-playbook -i inventory.ini capstone-services.yml` (no `--check`) against `client-vm2`.
2. Memory pre-flight check passed (3398MB, corrected 3072MB threshold from the Day 11 memory-threshold fix).
3. Directory creation and file copy tasks (`docker-compose.yml`, `.env`) succeeded.
4. `client-vm2` was automatically **paused** by VirtualBox mid-execution, with error: `Host system reported disk full. VM execution is suspended.`
5. Checked Windows host C: drive via Properties: **0 bytes free of 237GB capacity.**
6. Ran Windows Disk Cleanup, freed ~1.76GB by clearing temporary files, cache, and an unrelated large application (CapCut).
7. Resumed `client-vm2` successfully.
8. Checked guest-side disk (`df -h`): root filesystem showed 8.1G total, 6.9G used, **799M available** — tight but not itself the blocker; the guest disk didn't need to grow past its allocated size, the host simply couldn't allocate more space to the differencing disk file at all.
9. Checked `docker ps -a`: only a pre-existing, unrelated `promtail` container (exited 8 days prior) was present. **None of the four new capstone containers (wikijs, snipeit, snipeit-db, uptime-kuma) were created** — confirming the deploy failed cleanly during image pull, before any container state existed. No cleanup or rollback needed.

## Root Cause

The Windows host machine has a single 237GB C: drive, currently carrying the full weight of four VirtualBox VMs (`server-vm`, `client-vm2`, `freeipa-vm`, and the underlying `client-vm` base disk) plus all normal host applications and files. Pulling ~4-5 additional multi-hundred-MB Docker images simultaneously during the capstone deploy pushed `client-vm2`'s differencing disk file size past what the host had room to allocate.

This is a distinct issue from the Day 10 memory pressure incident and the Day 11 memory threshold correction — those were about RAM. This is disk capacity, and it sits at the host level, not something a VM-side Ansible check alone can fully guard against, since a `df -h` check inside the guest reports the guest's own filesystem, not whether the host has room to let a dynamically-sized virtual disk keep growing.

## Immediate Fix

Freed ~1.76GB on the host via Windows Disk Cleanup (temp files, cache, an unrelated large application). Enough to resume the paused VM and confirm no destructive state was left behind, but not enough to safely re-attempt the full multi-service deploy.

## Status: Day 11 Real-Deploy Validation Paused

The actual (non-check) deploy of the capstone services onto `client-vm2` was not completed. This is being logged as a deliberate pause rather than pushed through, because:
- Host free space (~1.76GB) is still far too tight to safely pull several more GB of Docker images
- Forcing another attempt risks repeating the same disk-full pause, or worse, corrupting VM state if it happens mid-write
- The underlying capacity problem (single 237GB host drive shared across 4 VMs) needs a real decision, not a stopgap cleanup

## Recommended Next Steps (before resuming Day 11)

1. Free significantly more host disk space — options include: uninstalling unused large applications, moving personal files (photos, videos, documents) to external/cloud storage, or reviewing whether all snapshot history across the VMs is still needed (old snapshots consume real disk space and accumulate over a multi-week project)
2. Consider whether all four VMs need to exist/run simultaneously going forward, or whether some (e.g. `freeipa-vm`) can be exported/archived when not actively in use
3. Once host has meaningfully more free space (recommend 15-20GB+ buffer), retry the real `capstone-services.yml` run against `client-vm2`
4. Consider adding a host-disk-awareness note to the playbook's documentation (not necessarily an automated check, since Ansible facts don't easily expose host-level free space from inside a guest) — at minimum, a manual pre-flight reminder in the runbook

## Lessons Learned

- **Multi-VM lab environments on a single-drive host have a real, finite ceiling that isn't visible from inside any one VM.** Guest-level disk and memory checks (like the Ansible `assert` added after Day 10) are necessary but not sufficient — they can't see host-level constraints.
- **A clean failure is a good failure.** Docker Compose pulling images and hitting a hard stop before creating any containers meant there was nothing to roll back or clean up — worth confirming this kind of graceful failure mode when designing future automation.
- **This is now the second host-resource-related incident in two days** (Day 10: RAM, Day 11: disk) — both stemming from the same underlying cause: an ambitious multi-service, multi-VM platform outgrowing conservative early-stage resource assumptions. Worth treating host capacity planning as its own explicit checklist item before Day 12+.

## Update: Repeated Deploy Attempts Confirm Root Cause, Session Paused

After the initial pause/resume/cleanup cycle documented above, the real (non-check) `capstone-services.yml` deploy was attempted twice more against `client-vm2`, once host free space had been increased via cleanup (`.cache` folder deletion, ~970MB) and a snapshot merge (`server-vm`'s `services-deployed-2026-08-04` snapshot deleted, recovering additional space, bringing host free space to 4.25GB at its peak).

**Attempt 2**: Failed with `unreachable: true` — Ansible lost SSH connectivity to `client-vm2` mid-task (`Shared connection to 10.10.10.20 closed`) during the `Bring up capstone services via Docker Compose` step, the same task that triggered the original disk-full pause. `client-vm2` remained running afterward but the Docker Compose operation did not complete.

**Result**: Host free space dropped from 4.25GB to 2.11GB over the course of this attempt — confirming that each retry consumes meaningful host disk space (partial Docker image layers pulled before failure) without completing successfully, even with a larger starting buffer than the initial attempt.

## Conclusion

Two consecutive real-deploy attempts, at different starting disk-space levels (470MB→1.76GB, then 4.25GB→2.11GB), both failed at the identical step. This confirms the root cause is not a transient/timing issue but a genuine, reproducible capacity shortfall: **the full capstone Docker image set requires more sustained free disk space than this host can currently provide**, even after two rounds of cleanup.

**Decision**: Stopped retrying for this session. `client-vm2` was left in a stable, running state with no corruption. Further attempts without a substantially larger host disk space intervention (5-10GB+ recovered, not the 1-2GB increments achieved so far) are expected to fail the same way and were judged not to be a good use of continued troubleshooting time in a single session.

## Revised Recommended Next Steps

The original recommendations stand, but with more urgency given two failed retries:
1. **Priority**: identify and remove/relocate large, non-essential items — candidates observed during this session include a second Cisco Packet Tracer installation (8.2.2 vs 9.0.0, if only one is in active use) and the `.ollama` folder (local AI model files, not yet sized but a common multi-GB consumer) — worth checking before the next attempt
2. Consider archiving/exporting `freeipa-vm` (15.2GB) if FreeIPA work is not immediately ongoing, to free that space until it's needed again
3. Do not attempt the real deploy again until host free space is confirmed at 10GB+, given the observed ~2GB consumption per failed attempt plus the actual space Docker images need to complete successfully
4. Once sufficient space is confirmed, retry `ansible-playbook -i inventory.ini capstone-services.yml` (no `--check`) as a single attempt rather than repeated tries, to avoid the same space-erosion pattern
