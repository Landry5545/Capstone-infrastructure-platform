# FreeIPA Installation: Platform Selection Saga

**Date:** August 2-3, 2026
**Related roadmap phase:** Day 4-6 — FreeIPA Identity Management
**VM:** `freeipa-vm` (10.10.10.30)

## Context

FreeIPA was placed on a dedicated third VM rather than `server-vm` to avoid a
DNS conflict: `server-vm` already runs BIND9 (from Project 4), and FreeIPA
either wants to run its own integrated DNS or needs careful coexistence
configuration. Isolating it onto `freeipa-vm` sidesteps that entirely.

## Attempt 1: Ubuntu 26.04, native package

Tried installing FreeIPA directly via `apt`. Result: the `freeipa-server`
package does not exist in the standard Ubuntu repositories. The community
FreeIPA PPA was tried as a fallback and found to be effectively abandoned —
zero updates in the past month, and no packages built for any current Ubuntu
release ("noble"/26.04 included).

**Conclusion:** Ubuntu is not a first-class FreeIPA platform, and the PPA
that used to bridge that gap is no longer maintained.

## Attempt 2: Docker-based FreeIPA on Ubuntu 26.04

Pivoted to the official `freeipa/freeipa-server:almalinux-9` Docker image,
since FreeIPA runs a full systemd init inside the container (LDAP, KDC,
Apache, etc. all as services). This required troubleshooting a chain of
systemd-in-container issues in sequence:

1. **cgroup read-only mount** — container couldn't get a writable cgroup
   hierarchy. Fixed by running with `--cgroupns=host`.
2. **cgroup v1/v2 mismatch** — the container's expected cgroup version
   didn't match the Ubuntu host's.
3. **Memory-check failure** — FreeIPA's install-time memory check failed
   inside the container. Worked around with `--skip-mem-check`.
4. **`hostnamectl`/D-Bus "Transport endpoint is not connected"** — an
   unresolvable error communicating with D-Bus inside the container,
   blocking hostname configuration. No viable fix found.

**Conclusion:** Running a full systemd-based service stack inside Docker on
a non-native host is fragile. Each fix uncovered the next layer of
incompatibility; abandoned after the D-Bus error proved unresolvable.

## Attempt 3: Ubuntu 24.04 LTS, hoping the PPA would help

Rebuilt `freeipa-vm` on Ubuntu 24.04 LTS on the chance the community PPA
supported it. Checked Launchpad directly and confirmed the PPA is abandoned
across the board — it does not support any current Ubuntu release,
24.04 included.

**Conclusion:** This wasn't an Ubuntu-version problem; the PPA itself is
dead. No native path to FreeIPA on Ubuntu currently exists.

## Final: AlmaLinux 9.8 (FreeIPA's native platform)

Pivoted to AlmaLinux 9.8 — a RHEL-compatible distribution where FreeIPA is a
first-class citizen with real `dnf` packages and no third-party PPA needed.

```bash
dnf install freeipa-server -y
```

installed cleanly with all dependencies (389 Directory Server, MIT Kerberos,
Apache/mod_wsgi, Dogtag CA, SSSD, Tomcat for the web console).

## Installation gotchas on AlmaLinux

Even on the right platform, two issues came up worth recording:

### 1. VirtualBox host disk full mid-install (`VERR_DISK_FULL`)

The Windows host ran out of disk space partway through package installation,
causing VirtualBox to throw `VERR_DISK_FULL` and pause the VM. Root cause:
leftover ISO files (Ubuntu 24.04 and 26.04, ~6 GB combined) from the earlier
failed attempts, still sitting in Downloads.

**Fix:** Freed host disk space (deleted the dead Ubuntu ISOs, emptied
Recycle Bin), then resumed the paused VM via VirtualBox Manager
(**Machine → Pause**, which toggles to resume when already paused).

**Post-incident verification** (to rule out a corrupted install):
```bash
dnf check                    # dependency/package DB consistency
rpm -Va --nodigest            # file-level verification across all packages
dnf update -y                 # real-world stress test
```
All three passed clean — the `rpm -Va` output showed only expected,
harmless diffs (authselect-managed PAM symlinks, SELinux context
timestamps, plymouth boot-duration files), and `dnf update` completed a
full kernel update without error.

### 2. `myhostname` in nsswitch.conf overriding `/etc/hosts`

After adding a `10.10.10.30 freeipa-vm.lab.local` entry to `/etc/hosts`,
`getent hosts freeipa-vm.lab.local` still returned only synthesized IPv6
link-local addresses, not the static IP. `ipa-server-install` depends on
this resolving correctly.

**Root cause:** `/etc/nsswitch.conf` had:
```
hosts:      files dns myhostname
```
The `myhostname` NSS module auto-generates addresses for the machine's own
hostname and was taking precedence over the `/etc/hosts` entry for
self-hostname lookups.

**Fix:**
```bash
# Edit /etc/nsswitch.conf, change:
hosts:      files dns myhostname
# to:
hosts:      files dns
```
After this, `getent hosts freeipa-vm.lab.local` correctly returned
`10.10.10.30`.

## Successful install

With both issues resolved, `ipa-server-install` completed successfully:

- Realm: `LAB.LOCAL`
- Domain: `lab.local`
- Integrated DNS: not configured (standalone, no conflict with `server-vm`'s
  BIND9)
- CA: self-signed, `CN=Certificate Authority,O=LAB.LOCAL`

## Post-install verification

```bash
kinit admin          # Kerberos authentication as IPA admin
klist                # confirmed valid ticket, krbtgt/LAB.LOCAL@LAB.LOCAL
```

Web UI reachability confirmed from `client-vm2` (after adding the same
`/etc/hosts` entry there):
```bash
curl -k -I https://freeipa-vm.lab.local/ipa/ui/
# HTTP/1.1 200 OK
```

Firewall opened for FreeIPA services:
```bash
firewall-cmd --permanent --add-service=freeipa-ldap
firewall-cmd --permanent --add-service=freeipa-ldaps
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload
```

## Backups / safety net

- VirtualBox snapshot taken: `freeipa-working-2026-08-03`
- CA certificate backed up: `/root/backups/cacert-20260803.p12`
  (required for future replica creation; password is the Directory
  Manager password)

## Key takeaway

Platform choice matters more than distribution familiarity. Three attempts
on Ubuntu (native package, Docker workaround, different Ubuntu version)
all failed for structural reasons — no native package, a dead PPA, and
systemd-in-Docker fragility — before pivoting to FreeIPA's actual
supported platform resolved everything in one clean install. The two
remaining issues (disk space, nsswitch ordering) were unrelated to the
Ubuntu/AlmaLinux question and would have surfaced on any platform.
