---
name: vps-provisioning
description: "[EXTRAS / ADVANCED] Use when provisioning a fresh VPS (Hetzner, Biznet, DigitalOcean, Ubuntu 24 LTS) for Kamal-based Rails deployments with advanced hardening. Applies to SSH lockdown, Tailscale private mesh, UFW firewall, Docker port discipline, Cloudflare-only HTTP/HTTPS via DOCKER-USER iptables, fail2ban, unattended upgrades, swap configuration, and Kamal deploy config. Triggers when user mentions advanced VPS hardening, Tailscale, iptables, DOCKER-USER, Cloudflare-only ingress, multi-server mesh, private SSH. Critical safety note: Docker bypasses UFW — you need DOCKER-USER iptables chain. For v1 simple setup, use vps-basics skill instead."
---

# VPS Provisioning & Hardening (Extras / Advanced)

> **This is an extras skill.** Most students should use `vps-basics` for v1 apps. Come back here when you need Tailscale private mesh, Cloudflare-only ingress, or multi-server networking.

> Step-by-step guide for advanced provisioning and securing of a VPS for deploying web apps with Docker, Kamal, Cloudflare, and Tailscale. Based on Ubuntu 24 LTS.

## 10 Principles

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

## ⚠️ Critical Safety Note

> **Docker bypasses UFW.** Docker injects iptables rules *before* UFW's chain. Any port in a `ports:` directive in docker-compose is accessible from the internet regardless of UFW rules. UFW only controls non-Docker services (SSH, systemd services, etc).

This is the single most important concept. Always use `DOCKER-USER` iptables chain for Docker-exposed ports.

## Provisioning Phases

### Phase 1: Initial Access & Lockdown
Get off root and off the public IP as fast as possible.

- Update system, install Docker + fail2ban + unattended-upgrades
- SSH keepalive config (prevent disconnects during long sessions)
- Create deploy user with SSH keys copied from root
- Passwordless sudo + docker group membership
- Install Tailscale, join tailnet
- **Verify Tailscale SSH works in separate terminal BEFORE locking down**
- UFW: deny incoming, allow only `100.64.0.0/10` (Tailscale CGNAT) to port 22
- Disable root login + password auth in sshd

### Phase 2: System Hardening
All commands run as deploy user via Tailscale SSH.

- Swap space (2G, swappiness=20) — prevents crashes during deploys
- `/storage` directory (700, owned by UID 1000) — for app data
- Enable fail2ban (default config protects SSH)
- Unattended upgrades config

### Phase 3: Docker Port Discipline (Most Critical)
Three patterns:

1. **Never expose internal services** — drop `ports:` for databases, services that only talk to other containers
2. **Bind to localhost for host-only** — `127.0.0.1:9000:9000` when only the host proxy needs access
3. **Host services consumed by containers** — keep on `0.0.0.0`, containers reach via `172.17.0.1`, UFW denies external

### Phase 4: Cloudflare-Only HTTP/HTTPS
Since kamal-proxy runs in Docker, UFW rules on 80/443 do nothing. Use DOCKER-USER iptables chain:

- Script at `/etc/docker/iptables-cloudflare.sh` fetches Cloudflare IPs from API
- Allows: established connections, Docker inter-container (172.17-172.19), Tailscale (100.64/10), loopback, Cloudflare IPs to 80/443
- Drops: everything else on 80/443
- Systemd service runs on boot + weekly timer refreshes Cloudflare IP ranges

### Phase 5: Kamal Deploy Config
Update `config/deploy.yml` to use Tailscale IP (not public IP):
```yaml
servers:
  web:
    - <vps-tailscale-ip>
```

Deploys now require Tailscale connected on developer's laptop.

## Three Common Pitfalls

### 1. Exposing databases via `ports:`
```yaml
# ❌ BAD — database exposed to internet
services:
  db:
    image: postgres:15
    ports:
      - "5432:5432"
```

Docker DNS handles service-to-service communication. Drop the `ports:` block.

### 2. Relying on UFW for Docker ports
UFW rules for ports 80/443 do nothing when Docker publishes those ports. Remove misleading UFW rules for Docker-exposed ports to avoid false sense of security:
```bash
sudo ufw delete allow 80
sudo ufw delete allow 443
```

### 3. Kamal-proxy losing DNS resolution
When running non-Kamal compose services alongside Kamal, `docker compose down && up` recreates the network and kamal-proxy returns 502s. Fix:
```yaml
services:
  app:
    networks:
      - default
      - kamal
networks:
  kamal:
    external: true
```

## Final State

| Port | Listening on | Reachable from internet | Controlled by |
|------|-------------|------------------------|---------------|
| 22 | 0.0.0.0 | No (Tailscale only) | UFW |
| 80/443 | 0.0.0.0 | Cloudflare IPs only | DOCKER-USER iptables |
| Databases | Not exposed | No | No `ports:` in docker-compose |
| Host services | 0.0.0.0 | No | UFW deny |

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

## Rollback Commands

| What | Command |
|------|---------|
| Restore public SSH | `sudo ufw allow 22` |
| Restore direct HTTP/S | `sudo iptables -F DOCKER-USER && sudo iptables -A DOCKER-USER -j RETURN` |
| Re-expose a Docker port | Add `ports:` back to docker-compose and `docker compose up -d` |
| Restore UFW allow for a port | `sudo ufw allow <port>/tcp` |

## When to Use

✅ **Use this skill when:**
- Setting up a fresh VPS for the first time (Hetzner, Biznet, DigitalOcean, etc.)
- Hardening an existing server (migrate to Tailscale-only SSH)
- Adding Cloudflare proxy + restricting direct IP access
- Running multi-app Kamal setup on one VPS
- Docker compose service exposing unintended ports
- Needing audit of current firewall + ports

❌ **Don't use when:**
- Shared hosting / PaaS (Heroku, Render, Fly.io) — platform handles infra
- Docker Swarm / Kubernetes — different network model
- Single-app setup with no external VPS needs

## Anti-Patterns

- ❌ Relying on UFW to protect Docker-published ports (doesn't work)
- ❌ Exposing databases with `ports:` — they reach each other via Docker DNS
- ❌ Using password-based SSH when keys + Tailscale are available
- ❌ Running everything as root instead of deploy user
- ❌ Leaving port 22 open to the world after Tailscale is installed
- ❌ Hardcoding Cloudflare IPs without refresh (they update)

## Key Concepts Reference

| Concept | What it means |
|---------|--------------|
| **Docker bypasses UFW** | Docker injects iptables rules before UFW's chain. UFW rules have no effect on Docker-published ports. Use `DOCKER-USER` chain instead. |
| **DOCKER-USER chain** | The only iptables chain Docker doesn't overwrite on restart. Rules here are evaluated before Docker's own chains. |
| **Tailscale CGNAT** | All Tailscale IPs are in `100.64.0.0/10`. One UFW rule covers all devices in your tailnet. |
| **Docker bridge gateway** | Containers reach the host at `172.17.0.1`. Binding a host service to `127.0.0.1` makes it unreachable from containers. |
| **Cloudflare proxy** | Orange-clouded domains route through Cloudflare. Restricting 80/443 to Cloudflare IPs ensures all traffic goes through their WAF/DDoS protection. |

## References

- `references/vps-provisioning-guide.md` — full step-by-step guide with all commands, SSH configs, iptables scripts, systemd service files, and case studies

## Related Skills

- **rails-debug-helper** — for deploy pre-flight checks (tests, DNS, Docker)
- **solid-suite-config** — for Solid Queue / Cable service configuration on deploy
