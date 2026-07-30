# Identity Management — Architecture Decision

**Project:** Design, Consolidation, and Extension of an Automated IT Infrastructure Platform for a Small Organization
**Component:** Centralized identity management (Day 4-6)

---

## 1. Why centralized identity at all?

Without it, each internal service (Wiki.js, Snipe-IT, Uptime Kuma) maintains its own separate user database. This means:
- Creating and managing every user account in multiple places independently
- No single point to disable an account when someone leaves
- Password policy, MFA, and permission audits all done N times over instead of once

Centralized identity management puts one authoritative source of truth: users authenticate once against a central directory, and every integrated service checks against that same source.

## 2. Options considered

### Plain LDAP (e.g. OpenLDAP)
Just a directory service — a hierarchical database of users and groups that can be queried and authenticated against.

- Gives the storage layer only. No built-in Kerberos (so no true single sign-on, only simple bind authentication), no certificate authority, no policy management UI, no DNS integration.
- All of that would need to be hand-built with custom schemas and scripts.
- Lightweight and well-understood, but a significant amount of manual plumbing for a project on a fixed timeline.

### Active Directory (Windows)
Microsoft's directory service, bundling LDAP, Kerberos, DNS, and Group Policy management.

- The enterprise standard, particularly in Windows-heavy environments.
- Requires Windows Server licensing and a Windows-centric administrative model (AD DS role, Group Policy Objects) — a different OS and tooling paradigm from the Linux/Docker/Ansible stack used throughout this project.
- Deliberately scoped out of this capstone as a separate, dedicated future project (see main README, Part 5 / Scope) rather than folded in here, to avoid a mid-project context switch across the whole toolchain.

### FreeIPA
Effectively "Active Directory for Linux" — bundles 389 Directory Server (LDAP), MIT Kerberos (SSO), Dogtag Certificate Authority, and optional integrated DNS, with web UI and CLI tooling included.

- Native to the Linux/Ubuntu environment already used throughout this project — no new OS, no new administrative paradigm.
- Provides Kerberos-based SSO out of the box, which plain LDAP does not, without the setup burden of assembling that stack manually.
- Conceptually comparable to Active Directory, making the "FreeIPA vs. AD" comparison a genuine, defensible design decision rather than a default choice.

## 3. Decision

**FreeIPA**, running on a dedicated third VM (`freeipa-vm`, 10.10.10.30), isolated from `server-vm`.

### Why a dedicated VM rather than sharing `server-vm`
- FreeIPA bundles its own integrated DNS component. Running it on `server-vm` — which already runs a hand-configured BIND9 zone (Project 4) — would require either letting FreeIPA take over DNS entirely, or fighting its installer to keep DNS management disabled, an under-documented and fragile mode.
- FreeIPA runs several resource-heavy daemons simultaneously (389 Directory Server, Dogtag CA, MIT Kerberos KDC). Co-locating these with `server-vm`'s existing load (Nginx, BIND9, the monitoring stack, the CI/CD runner) would create resource contention and make troubleshooting ambiguous — if something misbehaves, it wouldn't be clear whether the cause is FreeIPA or an existing service.
- Isolating identity management on its own host reflects standard practice: a security-sensitive service shouldn't share a blast radius with public-facing infrastructure like the reverse proxy.

## 4. Trade-offs and open questions

- FreeIPA's own DNS component is not used in this deployment (existing BIND9 on `server-vm` remains authoritative for the `lab.local` zone). This avoids conflict but means FreeIPA-specific DNS records (SRV records for Kerberos/LDAP discovery) will need manual configuration in BIND9 rather than being handled automatically.
- Full integration (at least one service authenticating against FreeIPA) is a stretch goal for this capstone rather than a hard requirement — see main README, Success Criteria.
