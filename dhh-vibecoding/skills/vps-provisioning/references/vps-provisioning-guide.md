> **Source:** Original VPS Provisioning & Hardening Guide by William Jakfar (BelajarGPT). Captures real patterns from multi-VPS Kamal deployments with Tailscale + Cloudflare. Preserved here verbatim for Claude Code reference.

---


# VPS Provisioning & Hardening Guide

A step-by-step guide for provisioning and securing a VPS for deploying web apps with Docker, Kamal, Cloudflare, and Tailscale. Based on Ubuntu 24 LTS.

## Principles

1. **No root SSH** — create a dedicated user, disable root login
2. **Key-only authentication** — no passwords over SSH
3. **Tailscale first** — install it early, lock down SSH immediately
4. **Deny by default** — UFW denies all incoming, allow only what's needed
5. **Never expose databases** — Docker services talk internally, not via host ports
6. **Cloudflare as the front door** — ports 80/443 only accept Cloudflare traffic
7. **Tailscale as the back door** — SSH and admin access only via private network
8. **Docker bypasses UFW** — use `DOCKER-USER` iptables chain for Docker ports
9. **Automatic security updates** — unattended upgrades for OS patches
10. **Brute force protection** — fail2ban for SSH and other services

---

## Phase 1: Initial Access & Lockdown

The goal is to get off root and off the public IP as fast as possible. All commands in this phase run as root over SSH — the first and last time.

### 1.1 Update System & Install Essentials

```bash
apt update
DEBIAN_FRONTEND=noninteractive apt upgrade -y
apt install -y docker.io curl unattended-upgrades fail2ban
```

### 1.2 SSH Keepalive

Prevent SSH disconnects during long provisioning sessions. Edit `/etc/ssh/sshd_config`:

```
ClientAliveInterval 60
ClientAliveCountMax 10
```

```bash
systemctl restart ssh
```

### 1.3 Create a Deploy User

```bash
useradd --create-home <username>
usermod -s /bin/bash <username>

# Copy SSH keys
su - <username> -c 'mkdir -p ~/.ssh'
su - <username> -c 'touch ~/.ssh/authorized_keys'
cat /root/.ssh/authorized_keys >> /home/<username>/.ssh/authorized_keys
chmod 700 /home/<username>/.ssh
chmod 600 /home/<username>/.ssh/authorized_keys

# Passwordless sudo
echo '<username> ALL=(ALL:ALL) NOPASSWD: ALL' | tee /etc/sudoers.d/<username>
chmod 0440 /etc/sudoers.d/<username>
visudo -c -f /etc/sudoers.d/<username>

# Add to docker group (so Kamal can manage containers without sudo)
usermod -aG docker <username>
```

> [!warning]
> Test SSH as the new user in a separate terminal **before** proceeding. If you lock yourself out, you'll need VPS console access to recover.

```bash
ssh <username>@<server-ip>
```

### 1.4 Install Tailscale & Lock Down SSH

Install Tailscale immediately — the less time port 22 is open to the world, the better.

```bash
# On VPS (still as root)
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
# Follow the auth URL to add to your tailnet
```

Make sure Tailscale is also installed on your laptop:

```bash
# macOS
brew install tailscale
# Or download from https://tailscale.com/download
```

Verify SSH works via Tailscale in a **separate terminal** before locking down:

```bash
# Check your tailnet — shows all devices with their Tailscale IPs and hostnames
tailscale status

# Test SSH via Tailscale IP
ssh <username>@<vps-tailscale-ip>

# Or use Tailscale hostname (e.g. ubuntu-8gb-hel1-1)
ssh <username>@<tailscale-hostname>
```

> [!warning]
> Do NOT close your current SSH session until you've confirmed Tailscale SSH works.

Once confirmed, lock down SSH and disable root:

```bash
# Restrict SSH to Tailscale only
ufw logging on
ufw default deny incoming
ufw default allow outgoing
ufw allow from 100.64.0.0/10 to any port 22
ufw --force enable

# Disable root login & password auth
sed -i 's@PasswordAuthentication yes@PasswordAuthentication no@g' /etc/ssh/sshd_config
sed -i 's@PermitRootLogin yes@PermitRootLogin no@g' /etc/ssh/sshd_config
systemctl restart ssh
```

`100.64.0.0/10` is the Tailscale CGNAT range — covers all devices in any tailnet.

From this point on, all access is via Tailscale as the deploy user. The public IP no longer accepts SSH.

**Rollback** (via Hetzner console if locked out): `ufw allow 22`

---

## Phase 2: System Hardening

All commands from here run as the deploy user via Tailscale SSH.

### 2.1 Swap Space

Prevents the application from crashing on low memory. Deploying new versions needs roughly double the memory (old + new container running simultaneously).

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab

# Lower swappiness — prefer RAM, use swap only under pressure
sudo sysctl vm.swappiness=20
echo 'vm.swappiness=20' | sudo tee -a /etc/sysctl.conf
```

### 2.2 Storage Directory

A generic `/storage` path for application data (uploads, backups, etc). Kamal accessories can mount this.

```bash
sudo mkdir -p /storage
sudo chmod 700 /storage
sudo chown 1000:1000 /storage
```

UID/GID 1000 is the first user created on the system (your deploy user).

### 2.3 Fail2ban

Brute force protection — bans IPs after repeated failed login attempts.

```bash
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
```

The default config already protects SSH. Check status with:

```bash
sudo fail2ban-client status sshd
```

### 2.4 Unattended Upgrades

Automatically installs security patches.

```bash
echo -e 'APT::Periodic::Update-Package-Lists "1";\nAPT::Periodic::Unattended-Upgrade "1";' | sudo tee /etc/apt/apt.conf.d/20auto-upgrades
sudo systemctl restart unattended-upgrades
```

> [!note]
> Unattended upgrades may cause server restarts. For production with zero-downtime requirements, consider removing servers from load balancers and performing manual updates instead.

---

## Phase 3: Docker Port Discipline

> [!important]
> **Docker bypasses UFW.** Docker injects iptables rules *before* UFW's chain. Any port in a `ports:` directive in docker-compose is accessible from the internet regardless of UFW rules. UFW only controls non-Docker services (SSH, systemd services, etc).

### 3.1 Never Expose Internal Services

If a Docker service only needs to talk to other containers in the same compose stack, do **not** add a `ports:` mapping. Docker's internal DNS handles service-to-service communication.

```yaml
# Bad — database exposed to the internet
services:
  db:
    image: postgres:15
    ports:
      - "5432:5432"    # Anyone on the internet can connect
  app:
    image: myapp
    environment:
      DATABASE_HOST: db  # Connects via Docker DNS anyway
```

```yaml
# Good — database only reachable within Docker network
services:
  db:
    image: postgres:15
    # No ports: block — only accessible from containers in this stack
  app:
    image: myapp
    environment:
      DATABASE_HOST: db
```

> [!tip] Case study
> Our Listmonk deployment had `ports: - "5432:5432"` on the PostgreSQL service, exposing the database to the entire internet. The app connected via Docker DNS (`host = "db"`) anyway — the port mapping was never needed. Removing it made the database invisible from outside.

### 3.2 Bind to Localhost When Only the Host Needs Access

If the host (not external traffic) needs to reach a container — e.g. a reverse proxy forwarding to it — bind to `127.0.0.1`:

```yaml
services:
  app:
    ports:
      - "127.0.0.1:9000:9000"  # Only reachable from the host
```

### 3.3 Host Services Consumed by Docker Containers

If a service runs on the host (systemd) and Docker containers need to reach it, you **cannot** bind to `127.0.0.1` — containers can't reach the host's loopback. Keep it on `0.0.0.0` and control access with UFW instead.

Docker containers reach the host via `172.17.0.1` (docker0 bridge gateway). UFW's INPUT chain doesn't filter this bridge traffic, so removing the UFW allow rule blocks external access while Docker containers remain unaffected.

```bash
# Remove the rule allowing public access
sudo ufw delete allow <port>/tcp

# UFW default deny blocks external traffic
# Docker bridge traffic (172.17.0.1) still works
```

> [!tip] Case study
> We run whisper-server as a systemd service on `0.0.0.0:8080`. A Docker container (todowill) calls it via `http://172.17.0.1:8080`. Removing `ufw allow 8080/tcp` blocked internet access while todowill continued working.

### 3.4 Kamal-Proxy and Docker Networks

When running non-Kamal Docker Compose services alongside Kamal-managed apps, kamal-proxy needs DNS resolution to reach containers by name. If you `docker compose down && up` a service, the Docker network is recreated and kamal-proxy loses resolution.

Fix: connect the container to kamal's network.

```yaml
services:
  app:
    networks:
      - default       # Internal compose network (for db, etc)
      - kamal         # Shared network with kamal-proxy

networks:
  kamal:
    external: true
```

> [!tip] Case study
> When we removed the PostgreSQL port from Listmonk's docker-compose and ran `docker compose down && up`, kamal-proxy returned 502 errors — it could no longer resolve `listmonk-app-1` because the Docker network had been recreated. Adding the `kamal` external network to the compose file fixed it permanently.

---

## Phase 4: Cloudflare-Only HTTP/HTTPS

All domains should be proxied (orange-clouded) through Cloudflare. Then restrict ports 80/443 to accept only Cloudflare IPs, blocking direct-IP access.

Since kamal-proxy is Docker, UFW rules on 80/443 do nothing. We use the `DOCKER-USER` iptables chain — the only chain Docker doesn't overwrite on restart.

### 4.1 Create the Firewall Script

File: `/etc/docker/iptables-cloudflare.sh`

```bash
#!/bin/bash
# Restrict Docker-exposed ports 80/443 to Cloudflare IPs only

set -euo pipefail

# Fetch Cloudflare IPs dynamically from their API (no auth required)
# Falls back to hardcoded list if API is unreachable
CLOUDFLARE_IPS=$(curl -sf --max-time 10 https://api.cloudflare.com/client/v4/ips | \
  python3 -c "import sys,json; r=json.load(sys.stdin); print(' '.join(r['result']['ipv4_cidrs']))" 2>/dev/null)

if [ -z "$CLOUDFLARE_IPS" ]; then
  echo "WARNING: API unreachable, using hardcoded fallback"
  CLOUDFLARE_IPS="173.245.48.0/20 103.21.244.0/22 103.22.200.0/22 103.31.4.0/22 141.101.64.0/18 108.162.192.0/18 190.93.240.0/20 188.114.96.0/20 197.234.240.0/22 198.41.128.0/17 162.158.0.0/15 104.16.0.0/13 104.24.0.0/14 172.64.0.0/13 131.0.72.0/22"
fi

# Flush DOCKER-USER (safe — only affects user rules)
iptables -F DOCKER-USER

# Allow established/related connections
iptables -A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN

# Allow Docker inter-container traffic (bridge networks)
iptables -A DOCKER-USER -s 172.17.0.0/16 -j RETURN
iptables -A DOCKER-USER -s 172.18.0.0/16 -j RETURN
iptables -A DOCKER-USER -s 172.19.0.0/16 -j RETURN

# Allow Tailscale subnet
iptables -A DOCKER-USER -s 100.64.0.0/10 -j RETURN

# Allow loopback
iptables -A DOCKER-USER -i lo -j RETURN

# Cloudflare IPv4 ranges — allow to ports 80,443
for ip in $CLOUDFLARE_IPS; do
  iptables -A DOCKER-USER -s "$ip" -p tcp -m multiport --dports 80,443 -j RETURN
done

# DROP everything else to 80,443
iptables -A DOCKER-USER -p tcp --dport 80 -j DROP
iptables -A DOCKER-USER -p tcp --dport 443 -j DROP

# RETURN everything else (don't interfere with other Docker traffic)
iptables -A DOCKER-USER -j RETURN

echo "Applied $(echo $CLOUDFLARE_IPS | wc -w) Cloudflare IP ranges"
```

```bash
sudo chmod +x /etc/docker/iptables-cloudflare.sh
```

### 4.2 Persist with Systemd Service + Weekly Timer

The iptables rules don't survive reboot. A systemd service applies them on boot, and a timer refreshes weekly in case Cloudflare adds new IP ranges.

File: `/etc/systemd/system/docker-cloudflare-fw.service`

```ini
[Unit]
Description=Apply Cloudflare IP restrictions to Docker
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
ExecStart=/etc/docker/iptables-cloudflare.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

File: `/etc/systemd/system/docker-cloudflare-fw.timer`

```ini
[Unit]
Description=Refresh Cloudflare IP restrictions weekly

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now docker-cloudflare-fw
sudo systemctl enable --now docker-cloudflare-fw.timer
```

### 4.3 Remove Misleading UFW Rules

If you had UFW rules for 80/443, remove them. They never actually protected Docker ports — keeping them creates a false sense of security.

```bash
sudo ufw delete allow 80
sudo ufw delete allow 443
```

### 4.4 Verify

```bash
# Direct IP access should time out
curl -k --connect-timeout 5 https://<server-ip>

# Domains via Cloudflare should still work
curl -s -o /dev/null -w "%{http_code}" https://yourdomain.com
```

**Rollback:** `sudo iptables -F DOCKER-USER && sudo iptables -A DOCKER-USER -j RETURN`

---

## Phase 5: Kamal Deploy Config

Update all `config/deploy.yml` files to use the Tailscale IP instead of the public IP:

```yaml
servers:
  web:
    - <vps-tailscale-ip>
```

Deploys now require Tailscale to be connected on your laptop.

> [!tip] Case study
> Our VPS (`37.27.181.71`) got Tailscale IP `100.78.134.64` and hostname `ubuntu-8gb-hel1-1`. We updated 5 Kamal deploy configs across different Rails apps. SSH via public IP now times out. All deploys and SSH go through Tailscale. Multiple VPSes each get their own Tailscale hostname — no need to remember IPs.

---

## Final State

After completing all phases:

| Port | Listening on | Reachable from internet | Controlled by |
|------|-------------|------------------------|---------------|
| 22 | 0.0.0.0 | No (Tailscale only) | UFW |
| 80/443 | 0.0.0.0 | Cloudflare IPs only | DOCKER-USER iptables |
| Databases | Not exposed | No | No `ports:` in docker-compose |
| Host services | 0.0.0.0 | No | UFW deny |

UFW status should show only:

```
22    ALLOW IN    100.64.0.0/10
```

System services running:

```
docker           — container runtime
fail2ban         — brute force protection
ufw              — firewall (non-Docker)
unattended-upgrades — automatic security patches
docker-cloudflare-fw.timer — weekly Cloudflare IP refresh
tailscaled       — Tailscale VPN daemon
```

## Verification Checklist

- [ ] `ssh root@<server-ip>` — permission denied
- [ ] `ssh <user>@<server-ip>` — timeout (public IP blocked)
- [ ] `ssh <user>@<tailscale-ip>` — works
- [ ] `ss -tlnp` — no database ports listening on host
- [ ] `curl -k https://<server-ip>` — timeout (direct IP blocked)
- [ ] All domains load via browser (Cloudflare routing works)
- [ ] `kamal deploy` works with Tailscale connected
- [ ] `sudo iptables -L DOCKER-USER -n` — shows Cloudflare IP rules
- [ ] `sudo systemctl list-timers` — shows weekly Cloudflare IP refresh
- [ ] `swapon --show` — shows 2G swap file
- [ ] `sudo fail2ban-client status sshd` — active with ban stats

## Key Concepts Reference

| Concept | What it means |
|---------|--------------|
| **Docker bypasses UFW** | Docker injects iptables rules before UFW's chain. UFW rules have no effect on Docker-published ports. Use `DOCKER-USER` chain instead. |
| **DOCKER-USER chain** | The only iptables chain Docker doesn't overwrite on restart. Rules here are evaluated before Docker's own chains. |
| **Tailscale CGNAT** | All Tailscale IPs are in `100.64.0.0/10`. One UFW rule covers all devices in your tailnet. |
| **Docker bridge gateway** | Containers reach the host at `172.17.0.1`. Binding a host service to `127.0.0.1` makes it unreachable from containers. |
| **Cloudflare proxy** | Orange-clouded domains route through Cloudflare. Restricting 80/443 to Cloudflare IPs ensures all traffic goes through their WAF/DDoS protection. |

## Rollback Commands

| What | Command |
|------|---------|
| Restore public SSH | `sudo ufw allow 22` |
| Restore direct HTTP/S | `sudo iptables -F DOCKER-USER && sudo iptables -A DOCKER-USER -j RETURN` |
| Re-expose a Docker port | Add `ports:` back to docker-compose and `docker compose up -d` |
| Restore UFW allow for a port | `sudo ufw allow <port>/tcp` |
