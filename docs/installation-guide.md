# Installation Guide

This guide walks through building the platform from scratch: three blank virtual machines to a fully working, monitored, CI/CD-deployed environment. It assumes no prior state — if you're troubleshooting an existing install instead, see [`troubleshooting-guide.md`](./troubleshooting-guide.md).

## Prerequisites

- A host machine running Oracle VirtualBox, with enough free disk space for four VM disks (recommend 60GB+ free before starting — see the troubleshooting guide's note on host disk capacity).
- Three VM disks planned at minimum: 40GB for `server-vm`, 20GB for `client-vm2`, 25GB for `freeipa-vm`. Undersizing these was a recurring source of incidents during development — do not start smaller than this.
- An Ubuntu Server 26.04 LTS ISO and an AlmaLinux 9.8 ISO downloaded to the host.
- A GitHub account with two repositories created: one for infrastructure documentation (`capstone-infrastructure-platform`) and one for Ansible playbooks (`ansible-lab`).

## 1. Network Planning

Create a VirtualBox **Internal Network** named `labnet` before building any VM. Every VM in this platform uses two adapters:

| Adapter | Type | Purpose |
|---|---|---|
| Adapter 1 | NAT | Independent internet access (package installs, Docker pulls, GitHub) |
| Adapter 2 | Internal Network: `labnet` | Internal-only traffic between the three VMs |

IP plan (static, assigned manually — not DHCP):

| Host | IP | Role |
|---|---|---|
| server-vm | 10.10.10.10 | Reverse proxy, DNS, monitoring, CI/CD, app containers |
| client-vm2 | 10.10.10.20 | Ansible-managed target, logging/metrics agents |
| freeipa-vm | 10.10.10.30 | Identity management |

## 2. Build server-vm

1. Create a VM: Ubuntu Server 26.04 LTS, 2 vCPU minimum, **3072MB RAM minimum** (see Section 8 below for why this specific figure matters), 40GB VDI, both adapters as above.
2. During install, set a static IP of 10.10.10.10/24 on the `labnet` adapter, with DNS pointed at itself (10.10.10.10) once BIND9 is installed in Step 4 — until then, use a public resolver temporarily.
3. After first boot, confirm SSH access and update the system:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
4. Configure the firewall baseline:
   ```bash
   sudo ufw allow 22/tcp
   sudo ufw allow 53
   sudo ufw allow 80/tcp
   sudo ufw allow 9100/tcp
   sudo ufw enable
   ```
5. Install Docker and Docker Compose:
   ```bash
   sudo apt install -y docker.io docker-compose-plugin
   sudo usermod -aG docker $USER
   ```
   Log out and back in for the group change to take effect.
6. Install BIND9:
   ```bash
   sudo apt install -y bind9 bind9utils
   ```
   Create a forward zone file at `/etc/bind/db.lab.local` with A records for `server`, `client`, `wiki`, `assets`, and `status`, all pointing at the appropriate internal IPs (service hostnames all resolve to 10.10.10.10, since Nginx routes them from there). Reference `/etc/bind/named.conf.local` to declare the zone, then:
   ```bash
   sudo systemctl restart bind9
   ```
7. Install Nginx:
   ```bash
   sudo apt install -y nginx
   ```
   **Do not assume this step is unnecessary because an earlier project covered it** — verify with `systemctl status nginx` before moving on, regardless of what prior documentation claims.
8. Confirm `nginx -t` passes before proceeding to service deployment.

## 3. Deploy the Three Internal Services

On server-vm, clone or create the Docker Compose project:

```bash
mkdir -p ~/capstone-infrastructure-platform && cd ~/capstone-infrastructure-platform
```

Create `docker-compose.yml` defining five services: Postgres (Wiki.js backend), Wiki.js (port 3001), MariaDB (Snipe-IT backend), Snipe-IT (port 3002), and Uptime Kuma (port 3003, self-contained, no separate database). Create a `.env` file (gitignored) with database credentials and app keys.

```bash
docker compose up -d
docker compose ps
```

Confirm all five containers show `Up`.

## 4. Configure Reverse Proxy Routing

For each service, create a site config under `/etc/nginx/sites-available/`:

```nginx
server {
    listen 80;
    server_name <hostname>.lab.local;
    location / {
        proxy_pass http://127.0.0.1:<port>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

For Uptime Kuma specifically, add WebSocket support (required for its live dashboard):
```nginx
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
```

Symlink each into `sites-enabled/`, test, and reload:
```bash
sudo ln -s /etc/nginx/sites-available/<file> /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Confirm all three are reachable locally:
```bash
curl -I http://wiki.lab.local
curl -I http://assets.lab.local
curl -I http://status.lab.local
```

Complete each service's first-run setup wizard (admin account creation). If the VM you're browsing from has no GUI browser, add the hostnames to your host machine's own hosts file instead and browse from there.

## 5. Install the Monitoring and Logging Stack

Add Prometheus, Grafana, node_exporter, and cAdvisor as additional Docker Compose services on server-vm, each with their own config/volumes. No special integration is required for cAdvisor to pick up the application containers — it auto-discovers everything on the Docker host. Add Uptime Kuma monitors for each of the three service URLs once Uptime Kuma itself is reachable.

For Snipe-IT's monitor specifically, set **Max Redirects to 0** — the container's internal redirect isn't reachable from inside the monitor's network context and will otherwise report a false failure.

## 6. Build client-vm2

1. Create a VM: Ubuntu Server, 2 vCPU, 4GB RAM, 20GB VDI, both adapters.
2. Set a static IP of 10.10.10.20/24, with DNS pointed at 10.10.10.10 (server-vm).
3. **Check `/etc/netplan/*.yaml` after applying the static config** — cloud-init sometimes generates a `nameservers:` block pointing at public DNS that silently overrides the intended server, even when the rest of the config looks correct.
4. Install and run node_exporter and Promtail as Docker containers (not the deprecated standalone Promtail binary), pointed at server-vm's Prometheus/Loki endpoints.
5. Confirm DNS resolution works both ways:
   ```bash
   ping -c 2 server.lab.local
   nslookup wiki.lab.local
   ```

## 7. Build freeipa-vm

FreeIPA has no native, supported package on Ubuntu — install **AlmaLinux 9.8** for this VM specifically, not Ubuntu. Attempting FreeIPA on Ubuntu (natively or in Docker) is a known dead end; see the troubleshooting guide.

1. Create a VM: AlmaLinux 9.8, 2 vCPU, 4GB RAM, 25GB VDI, both adapters.
2. Set a static IP of 10.10.10.30/24 via `nmcli` (NetworkManager, not Netplan — different OS), with DNS pointed at 10.10.10.10.
3. Install and run the installer:
   ```bash
   sudo dnf install -y freeipa-server
   sudo ipa-server-install
   ```
   Use realm `LAB.LOCAL`, domain `lab.local`, hostname `freeipa-vm.lab.local`. **Choose standalone mode — disable FreeIPA's own integrated DNS** when prompted, to avoid conflicting with BIND9 on server-vm.
4. Verify:
   ```bash
   kinit admin
   klist
   ```
5. Create at least one test user and group to confirm the directory itself works end to end.

## 8. Automate with Ansible

On server-vm (or a dedicated control node), install Ansible and create an inventory covering server-vm (localhost) and client-vm2 as the managed group. Write a playbook capturing everything done manually in Steps 2–5: the memory pre-flight check, the Docker Compose deploy, the Nginx configs, the BIND9 zone, and the UFW rule.

**On the memory pre-flight check specifically:** if you assert a minimum RAM threshold, set it to roughly 700MB below the VM's actual configured VirtualBox RAM, not equal to it — the guest OS reports less than the configured figure due to normal overhead, and an exact-match threshold will fail on hardware that is actually fine.

Validate with a dry run before a real deploy:
```bash
ansible-playbook -i inventory.ini capstone-services.yml --check
```

Commit every file the playbook references under `files/` to version control — including non-secret configs like the Nginx site files and the BIND9 zone file. A playbook that works against the server it was written on is not proof the repository itself is complete; only a clean clone, run fresh, proves that.

## 9. Extend CI/CD

1. Register a self-hosted GitHub Actions runner on server-vm for the `ansible-lab` repository, and install it as a systemd service so it survives reboots:
   ```bash
   mkdir ~/actions-runner-ansible-lab && cd ~/actions-runner-ansible-lab
   # download + extract runner package per GitHub's own instructions (token expires in ~1 hour, use it promptly)
   ./config.sh --url https://github.com/<you>/ansible-lab --token <TOKEN>
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```
2. Create 5 GitHub repository secrets for the database/app credentials referenced in `.env`.
3. Write a workflow (`.github/workflows/deploy-capstone.yml`) that triggers on push to `capstone-services.yml` or `inventory.ini`, reconstructs `.env` from the secrets at runtime, and runs the Ansible playbook via `runs-on: self-hosted`.
4. Test with a trivial push and confirm a green run in the repo's Actions tab before relying on it.

## 10. Final Verification

Before considering the install complete, confirm all of the following:

- [ ] All three services reachable by hostname through Nginx from both VMs
- [ ] `docker compose ps` shows every container `Up` on server-vm
- [ ] Prometheus, Grafana, cAdvisor, and node_exporter all showing data for both VMs
- [ ] Uptime Kuma monitors for all three services passing
- [ ] `kinit admin` succeeds on freeipa-vm
- [ ] `ansible-playbook ... --check` passes cleanly against both hosts
- [ ] A real (non-check) Ansible run completes with `failed=0`
- [ ] A test `git push` triggers a green CI/CD run end to end
