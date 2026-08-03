# Automated IT Infrastructure Platform

**Design, Consolidation, and Extension of an Automated IT Infrastructure Platform for a Small Organization**

## Overview

The IT department of a small organization lacks a centralized, automated, and monitored platform for deploying and managing internal services. Manual deployment is time-consuming, difficult to reproduce, and hard to maintain.

This project builds an automated infrastructure capable of deploying, securing, monitoring, and managing a small organization's IT services using Infrastructure as Code and containerization — consolidating nine standalone lab projects into a single documented platform and extending it with identity management and real internal services.

## Foundation — Prior Projects

This platform builds on nine sequential lab projects, each in its own repository:

| # | Project | Repo |
|---|---|---|
| 1 | Linux networking lab (static IPs, SSH keys, UFW) | [`linux-networking-lab`](https://github.com/Landry5545/linux-networking-lab) |
| 2 | Dockerized web infrastructure | [`dockerized-web-infrastructure`](https://github.com/Landry5545/dockerized-web-infrastructure) |
| 3 | Nginx reverse proxy | [`nginx-reverse-proxy`](https://github.com/Landry5545/nginx-reverse-proxy) |
| 4 | Custom DNS server (BIND9) | [`custom-dns-server`](https://github.com/Landry5545/custom-dns-server) |
| 5 | Monitoring stack (Prometheus, Grafana, cAdvisor, node_exporter) | [`monitoring-stack`](https://github.com/Landry5545/monitoring-stack) |
| 6 | GitHub Actions CI/CD with self-hosted runner | [`monitoring-stack`](https://github.com/Landry5545/monitoring-stack) |
| 7 | Ansible automation | [`ansible-lab`](https://github.com/Landry5545/ansible-lab) |
| 8 | Docker + UFW hardening (`DOCKER-USER` chain) | [`ansible-lab`](https://github.com/Landry5545/ansible-lab) |
| 9 | Loki + Promtail centralized logging | [`monitoring-stack`](https://github.com/Landry5545/monitoring-stack) |

All nine were re-verified from a fresh state as part of this capstone's Day 1 audit — see [`audit/day1-findings.md`](./audit/day1-findings.md).

## Infrastructure

| Host | Role | IP |
|---|---|---|
| `server-vm` | Reverse proxy, DNS, monitoring, CI/CD runner | 10.10.10.10 |
| `client-vm` | Node exporter, log shipping, test workloads | 10.10.10.20 |
| `freeipa-vm` | Identity management (FreeIPA) | 10.10.10.30 |
| Internal network | `labnet` | — |

## New Additions (Capstone Scope)

- **FreeIPA** — centralized identity management
- **Wiki.js** — internal documentation
- **Snipe-IT** — IT asset management
- **Uptime Kuma** — external service monitoring

All three services will be deployed behind the existing Nginx reverse proxy, resolvable via internal BIND9 DNS entries, integrated into the existing Prometheus/Grafana/Loki monitoring stack, provisioned via Ansible, and deployed through the existing GitHub Actions CI/CD pipeline.

## Technology Stack

| Layer | Technology |
|---|---|
| OS / networking | Ubuntu Server, static IP, SSH, UFW |
| DNS | BIND9 |
| Containers | Docker, Docker Compose |
| Reverse proxy | Nginx |
| Metrics | Prometheus, Grafana, node_exporter, cAdvisor |
| Logging | Loki, Promtail |
| Identity | FreeIPA (AlmaLinux 9.8) |
| Automation | Ansible |
| CI/CD | GitHub Actions |

## Repository Structure

```
capstone-infrastructure-platform/
├── audit/            # Infrastructure audit findings (Day 1+)
├── ansible/           # Consolidated playbooks for new services
├── docker-compose/    # Compose files for Wiki.js, Snipe-IT, Uptime Kuma
├── docs/              # Architecture diagrams, troubleshooting guide
└── .github/workflows/ # CI/CD pipeline for new services
```

## Scope

**Included:** Ubuntu Server, Docker/Compose, Nginx, BIND9, Prometheus/Grafana/Loki, Ansible, GitHub Actions, FreeIPA, three internal services, full documentation.

**Not included:** Kubernetes, multi-node clustering, cloud deployment, high availability, production-grade CA (self-signed/internal CA is acceptable for this lab). SSL/TLS, WireGuard VPN, and Windows Active Directory are treated as separate follow-on projects.

## Status

🟢 Day 1 — Infrastructure audit complete
⬜ Day 2-3 — Hardening
🟢 Day 4-6 — FreeIPA identity management complete — see [`audit/freeipa-installation-saga.md`](./audit/freeipa-installation-saga.md)
⬜ Day 7-9 — Service deployment
⬜ Day 10 — Unified monitoring/logging
⬜ Day 11 — Ansible consolidation
⬜ Day 12 — CI/CD extension
⬜ Day 13-14 — Unified architecture documentation
⬜ Day 15-16 — Report and presentation

## License

MIT — see [`LICENSE`](./LICENSE)
