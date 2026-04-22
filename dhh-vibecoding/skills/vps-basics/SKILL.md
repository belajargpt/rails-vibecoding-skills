---
name: vps-basics
description: Use when setting up a fresh VPS for the first time to host a Rails app via Kamal. Covers minimum viable provisioning — user creation, SSH key auth, firewall (UFW), swap, Docker install, fail2ban. Deliberately avoids Tailscale / Cloudflare / iptables complexity (see vps-provisioning extras skill for that). Triggers when user mentions new VPS, DigitalOcean droplet, Contabo, Hetzner, Biznet Gio, setting up a server, SSH setup, UFW, firewall, swap, Docker install, deploy user, fail2ban, hardening basics, or "prepare server for Kamal".
---

# VPS Basics — Minimum Viable Provisioning

> Get a fresh Ubuntu VPS ready for `kamal setup` in 15 minutes. No Tailscale, no Cloudflare, no iptables jungle. Just enough security for a solo builder's v1 app.

## Philosophy

> Simple beats clever. A public VPS with SSH key auth + UFW + fail2ban is secure enough for v1. Layer complexity later only when actual scale or threat model demands it.

Upgrade to `vps-provisioning` (extras skill) when you need:
- Private-only admin access (Tailscale)
- Cloudflare-only ingress (iptables)
- Multi-server mesh networking

For now — public server, locked down reasonably, ready to deploy.

## Target Setup

- **Provider:** DigitalOcean, Contabo, Hetzner, Biznet Gio, or any Ubuntu VPS
- **OS:** Ubuntu 24.04 LTS (or 22.04 LTS)
- **Size:** 2GB RAM minimum (4GB recommended for Rails + Postgres + Kamal Proxy)
- **Access:** Public IP, SSH on port 22

## The 15-Minute Checklist

1. Update system
2. Create deploy user
3. SSH key auth (disable password)
4. UFW firewall
5. Swap file (2GB)
6. Docker install
7. fail2ban
8. Verify + ready for `kamal setup`

## Step-by-Step

### 1. Log in as root (first time)

```bash
ssh root@<server-ip>
```

### 2. Update system

```bash
apt update && apt upgrade -y
apt install -y curl wget git vim ufw fail2ban
```

### 3. Create deploy user

Non-root user for everything Kamal-related:

```bash
adduser deploy
usermod -aG sudo deploy
```

Set password (or skip — we'll disable password auth anyway).

### 4. SSH key auth for deploy user

On your **laptop** (not server):

```bash
# If you don't have a key yet
ssh-keygen -t ed25519 -C "you@laptop"

# Copy to server
ssh-copy-id deploy@<server-ip>
```

Test:
```bash
ssh deploy@<server-ip>   # should log in without password
```

### 5. Disable password auth + root SSH

On the **server**:

```bash
sudo vim /etc/ssh/sshd_config
```

Set these values:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Reload SSH:
```bash
sudo systemctl restart ssh
```

**Critical:** Keep your current root SSH session open while testing `ssh deploy@<server-ip>` from another terminal. If deploy login fails, you can still fix via root session.

### 6. UFW firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP (Kamal Proxy + Let's Encrypt)
sudo ufw allow 443/tcp     # HTTPS

sudo ufw enable
sudo ufw status verbose
```

**Why ports 80 + 443:** Kamal Proxy listens on both. Let's Encrypt needs port 80 for cert challenge.

### 7. Swap file (2GB)

Rails + Docker on 2GB RAM VPS needs swap to avoid OOM:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Persist across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify
free -h
```

### 8. Install Docker

```bash
# Official Docker install script
curl -fsSL https://get.docker.com | sudo sh

# Allow deploy user to run Docker without sudo
sudo usermod -aG docker deploy

# Verify
sudo docker version
```

Log out + back in for group change to apply:
```bash
exit
ssh deploy@<server-ip>
docker ps   # should work without sudo
```

### 9. fail2ban (ban brute-force attempts)

Default config is fine for SSH:

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check it's watching SSH
sudo fail2ban-client status sshd
```

Ban threshold: 5 failed attempts → 10-minute ban. Adjust in `/etc/fail2ban/jail.local` if needed.

### 10. Verify — ready for Kamal

From your laptop:
```bash
ssh deploy@<server-ip> "docker version && echo 'READY'"
```

If you see Docker version + `READY`, server is good.

## Now Deploy via Kamal

Back on laptop in your Rails project:
```bash
# config/deploy.yml already has:
# servers:
#   web:
#     - <server-ip>
# ssh:
#   user: deploy

kamal setup
```

See `kamal-deployment` skill for full Kamal config.

## What We Skipped (and When to Add It)

| Skipped | Why | Add when |
|---|---|---|
| **Tailscale** | Complex setup, requires account | You want private-only SSH access |
| **Cloudflare tunnel / iptables** | Advanced firewall rules | Public IP abuse (DDoS, scanning) |
| **unattended-upgrades** | Nice-to-have | Long-running prod server |
| **Monitoring (Prometheus, Netdata)** | Over-engineered for v1 | Multiple apps or oncall rotation |
| **Backup strategy** | Kamal + git + DB dumps cover v1 | Revenue-critical data |
| **fail2ban rules beyond SSH** | Default is enough | Abuse patterns emerge |

See `vps-provisioning` skill (extras) for Tailscale + Cloudflare hardening.

## Common Gotchas

### "Permission denied (publickey)" after disabling password auth
- Did `ssh-copy-id` succeed before you disabled password auth? Re-enable temporarily via root session, re-copy key.

### Docker install fails with "package not found"
- Old Ubuntu version. Stick to 22.04 or 24.04 LTS.

### UFW blocks Docker containers
- Docker manages its own iptables rules. If you see issues, check `iptables -L` — Docker chains should be there.
- Fix: edit `/etc/default/ufw` → `DEFAULT_FORWARD_POLICY="ACCEPT"`, then `sudo ufw reload`.

### Out of memory during `kamal deploy`
- 2GB RAM too small for Rails build. Either:
  - Add swap (step 7 covers 2GB; add 4GB for tight VPS)
  - Build image on laptop or CI, let server just pull

### "Let's Encrypt rate limited"
- Cert requests hit Let's Encrypt staging first during debug.
- Fix: in `config/deploy.yml` under `proxy:`, set `ssl: true` only when DNS is verified.

## VPS Provider Quick Picks

| Provider | Cost (2GB) | Region | Notes |
|---|---|---|---|
| **DigitalOcean** | $12/mo | Global | Best docs, easy for beginners |
| **Hetzner** | €4/mo | EU/US | Cheapest reliable |
| **Contabo** | €5/mo | EU/US/Asia | Great value, slower IO |
| **Biznet Gio** | Rp 150K/mo | Jakarta | Best Indonesia latency |
| **Linode / Akamai** | $12/mo | Global | Solid alternative to DO |

For Indonesian market (course audience): **Biznet Gio** for Indonesia-first apps, **DigitalOcean SGP1** for global apps.

## Anti-Patterns

- ❌ Deploying Kamal as root (use deploy user)
- ❌ Leaving password auth enabled (SSH key only)
- ❌ Opening all ports "just in case" (deny by default, allow specific)
- ❌ No swap on small VPS (Docker OOMs during build)
- ❌ Skipping fail2ban (SSH bots hammer port 22 constantly)
- ❌ Running `apt upgrade` during traffic hours (can restart services)
- ❌ Storing server passwords in 1Password but not SSH keys (key is the access; password is dead)

## When to Upgrade to `vps-provisioning` (Extras)

Move to advanced hardening when:
- You have real users (privacy / compliance matters)
- You're seeing bot abuse (logs full of 404 scans)
- You want zero public SSH exposure (Tailscale private mesh)
- You're running multiple apps across servers (need mesh networking)

For a v1 app with < 100 users — `vps-basics` is sufficient.

## Related Skills

- **kamal-deployment** — deploy Rails apps to this provisioned VPS
- **vps-provisioning** (extras) — Tailscale + Cloudflare + iptables hardening
- **rails-debug-helper** — production log reading, deploy pre-flight
