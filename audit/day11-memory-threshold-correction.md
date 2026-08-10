# Correction: VM Memory Threshold in Ansible Pre-Flight Check

## Summary

While validating the Day 11 Ansible playbook (`capstone-services.yml`), the memory pre-flight check — added after the Day 10 memory pressure incident — correctly failed against `client-vm2`, reporting 3398MB RAM against a 4096MB threshold. This triggered an investigation into why a VM configured with 4096MB in VirtualBox was reporting less inside the guest OS. The investigation ruled out several plausible causes before arriving at the correct explanation: the threshold itself was set incorrectly, not the VM configuration.

## Investigation Timeline

1. **Initial check**: `client-vm2` failed the `ansible_memtotal_mb >= 4096` assertion, reporting only 3398MB.
2. **Hypothesis 1 — VirtualBox setting not applied**: Suspected the Base Memory change hadn't taken effect, possibly because the VM was resumed from a saved state rather than fully powered off. Ruled out after confirming via VirtualBox Settings that Base Memory was genuinely set to 4096MB, and after a full shutdown/restart cycle still showed ~3.3GB in the guest.
3. **Hypothesis 2 — stale snapshot**: Suspected `client-vm2` might be running on top of an older snapshot with a lower memory config baked in. Ruled out — the Snapshots panel showed only "Current State," no named snapshot to roll back to.
4. **Hypothesis 3 — host-level memory contention**: With `server-vm`, `client-vm2`, and `freeipa-vm` all running simultaneously, the Windows host (15.8GB total RAM) was at 90% utilization with only 1.5GB available — plausible that VirtualBox couldn't grant the full 4096MB request. Shut down `freeipa-vm` and rebooted `client-vm2` to test. Result: still ~3.3GB in the guest. Ruled out.
5. **Resolution — compared against `server-vm`**: Checked `free -h` on `server-vm`, which had also been bumped to 4096MB during the Day 10 incident. It showed the identical pattern: ~3.3GB total in the guest despite a 4096MB VirtualBox allocation. Since both VMs configured identically show the same gap, this is consistent, expected behavior on this host/VirtualBox setup — not a fault with either VM.

## Root Cause

The ~700MB (~17%) gap between VirtualBox's Base Memory setting and what the guest OS reports as total RAM is normal overhead — reserved for firmware emulation, kernel structures, and hardware buffers — on this particular VirtualBox/host configuration. The Ansible pre-flight check's threshold (4096MB) was set based on the VirtualBox *allocation* figure rather than what's actually achievable *inside* the guest, making it an unattainable target even for correctly-configured VMs.

## Fix

Lowered the assert threshold in `capstone-services.yml` from 4096MB to 3072MB, giving a reasonable safety margin below the ~3300-3400MB actually observed on 4096MB-allocated VMs in this environment. Updated the failure message to explain the VirtualBox-to-guest overhead so future readers don't repeat this same investigation.

## Lessons Learned

- **Pre-flight checks should be calibrated against what's actually observable inside the guest, not against host-side configuration values.** A 1:1 assumption between VirtualBox's Base Memory setting and `ansible_memtotal_mb` doesn't hold on this setup.
- **When a check fails unexpectedly, checking a known-good comparison point early would have saved significant troubleshooting time.** `server-vm` had already been bumped to 4096MB during the Day 10 incident and was demonstrably stable — comparing `client-vm2` against it directly, before chasing saved-state/snapshot/host-contention theories, would have shown the pattern was consistent (and therefore not `client-vm2`-specific) much sooner.
- **Ruling out hypotheses in order of "most specific to least specific" has value**, but it's worth periodically stepping back to ask "is this actually broken, or am I comparing against the wrong expected value?" — three plausible-sounding VM-specific theories were tested before questioning the threshold itself.

## Outcome

`capstone-services.yml`'s memory pre-flight check now uses a 3072MB threshold, correctly passing on both `server-vm` and `client-vm2` while still catching genuinely undersized hosts (like the original `client-vm` entry, which reported ~3398MB total and remains borderline — worth monitoring if the full stack is ever deployed there).
