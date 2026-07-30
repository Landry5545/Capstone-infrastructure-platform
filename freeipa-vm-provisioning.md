# FreeIPA VM Provisioning — Findings & Troubleshooting Log

**Project:** Design, Consolidation, and Extension of an Automated IT Infrastructure Platform for a Small Organization
**Date:** 2026-07-25 to 2026-07-30
**Scope:** Provisioning a dedicated third VM (`freeipa-vm`) for FreeIPA identity management, isolated from `server-vm` to avoid conflicting with the existing BIND9 DNS setup.

---

## 1. Design decision — dedicated VM vs. shared host

Before provisioning, considered whether FreeIPA should run on `server-vm` (alongside Nginx, BIND9, the monitoring stack, and the CI/CD runner) or on its own VM.

**Decision: dedicated VM.** Rationale:
- FreeIPA bundles its own integrated DNS component, which would conflict with the existing hand-configured BIND9 zone on `server-vm` (Project 4) unless explicitly disabled — a fragile, poorly-documented mode to rely on.
- FreeIPA runs several resource-heavy daemons (389 Directory Server, Dogtag CA, MIT Kerberos KDC), which would compete for resources with `server-vm`'s existing workload.
- Isolating identity management on its own host is standard practice — a security-sensitive service shouldn't share a blast radius with public-facing infrastructure like the reverse proxy.

## 2. VM specification

| Setting | Value |
|---|---|
| Name | `freeipa-vm` |
| OS | Ubuntu Server 26.04 LTS |
| vCPUs | 2 |
| RAM | 4096 MB |
| Disk | 25 GB, dynamically allocated VDI |
| Adapter 1 | NAT (internet access) |
| Adapter 2 | Internal Network, `labnet` |
| Static IP | 10.10.10.30/24 |
| Gateway / DNS | 10.10.10.10 (`server-vm`) |

## 3. Incident — host disk exhaustion during install (`VERR_DISK_FULL`)

### What happened
Partway through the Ubuntu Server installation (mid-partition-write), the install failed with:
```
The I/O cache encountered an error while updating data in medium "ahci-0-0" (rc=VERR_DISK_FULL).
```

### Root cause
The Windows host's C: drive had **0 bytes free of 237 GB**. This was not caused by the new VM's disk allocation — investigation (via WinDirStat) showed VirtualBox VDI files across all VMs (`server-vm`, `client-vm2`, `freeipa-vm`, plus one orphaned `client-vm` folder) totaled only ~20 GB combined, ruling out VM storage as the cause.

The actual space was consumed by:
| Source | Size |
|---|---|
| `Windows` folder | ~66 GB (fluctuated during cleanup) |
| `Program Files` + `Program Files (x86)` | ~75 GB combined |
| `Users` | ~66 GB |
| `pagefile.sys` | ~27 GB |

The single largest reclaimable item was having **both Visual Studio Community 2019 and 2022** installed simultaneously — an unused leftover from an earlier, unrelated install.

### Resolution
1. Emptied Recycle Bin (files deleted during initial cleanup attempts had not actually freed space until this step — moved to Recycle Bin, not removed)
2. Uninstalled Visual Studio Community 2019 via the Visual Studio Installer (not in use; VS 2022 retained)
3. Verified via WinDirStat before taking further action, to avoid deleting anything still in use

**Result:** free space went from 0 bytes → 24.13 GB, enough headroom to complete the install.

### Near-miss avoided
During cleanup, an orphaned-looking folder `client-vm` (6.85 GB) was initially flagged for deletion as leftover from a prior `.vbox` config corruption (documented separately from Project 7 — the original `client-vm`'s machine config file was corrupted, and a new VM `client-vm2` was created rather than repairing it).

Before deleting, verification via VirtualBox's Storage settings and Virtual Media Manager confirmed **`client-vm2`'s actual disk was the same physical `client-vm.vdi` file** in that "orphaned" folder — the new VM had never been given its own disk; it referenced the original. Deleting the folder would have destroyed the active, in-use VM's disk.

**Lesson documented:** folder names and VM names in VirtualBox are independent — a VM's registered name does not necessarily match its underlying file names, especially after a recreated/renamed VM. Always confirm the actual file path via VirtualBox's own Storage/Media Manager before deleting anything based on folder naming alone.

## 4. Network configuration gap

After the successful install, an unrelated host power loss caused an ungraceful VM abort during the install's network configuration step. On next boot, `enp0s8` (the `labnet`-facing adapter) was up at the link layer but had no IPv4 address assigned — the static IP step had been skipped.

**Resolution:** Configured directly via Netplan post-install rather than re-running the installer:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses:
        - 10.10.10.30/24
      routes:
        - to: default
          via: 10.10.10.10
      nameservers:
        addresses:
          - 10.10.10.10
```
Applied with `sudo netplan apply`. Confirmed via `ip a` and `ping -c 4 10.10.10.10` (0% packet loss, ~1ms latency).

## 5. Summary

| # | Issue | Resolution |
|---|---|---|
| 1 | Host disk full (`VERR_DISK_FULL`) during install | Freed 24+ GB via Recycle Bin cleanup and removing unused Visual Studio 2019 |
| 2 | Near-deletion of active VM disk (`client-vm2`) | Caught via Virtual Media Manager verification before deleting |
| 3 | `labnet` adapter missing static IP after ungraceful shutdown | Manually configured via Netplan, verified with ping test |

**Outcome:** `freeipa-vm` is provisioned, networked, and reachable at 10.10.10.30. Ready to proceed with FreeIPA package installation.
