---
name: kamal-deployment
description: Use when deploying Rails/Ruby/containerized apps to your own VPS with Kamal 2. Applies to kamal init, kamal setup, kamal deploy, kamal rollback, kamal app logs, kamal app exec, deploy.yml configuration, Kamal Proxy, accessories (databases/Redis), roles (web/job), hooks (pre-deploy, post-deploy), secrets management, multi-server deploys, multi-app on single VPS (subdomain routing), SSL/TLS via Let's Encrypt, zero-downtime deploys, rollbacks. Triggers when user mentions Kamal, kamal deploy, deploy.yml, zero-downtime, blue-green deploy, Docker deploy, reverse proxy, kamal-proxy, kamal accessory, kamal secrets, Capistrano alternative, Heroku alternative.
---

# Kamal Deployment

> Kamal 2 — imperative deployment tool by 37signals. Replaces Capistrano for the container era. Zero-downtime deploys via blue-green on containers + Kamal Proxy.

## Philosophy

> Push model: Kamal on your laptop/CI builds images and instructs servers. No agents on servers. No Kubernetes. Just Docker + SSH + Kamal Proxy.

**Key traits:**
- **Push model** — you control when deploys happen
- **Zero-downtime** — new container boots and passes health check before old is stopped
- **Simple components** — SSH (via SSHKit), Docker, Kamal Proxy
- **Language agnostic** — deploys any containerized app
- **Not Kubernetes** — simpler, tradeoff: no auto-scaling, no self-healing

## Core Concepts

| Term | Meaning |
|---|---|
| **service** | The name of your app (e.g., `my-blog`) |
| **server** | A virtual/physical host running your container |
| **role** | Server group with same command: `web` (primary, serves HTTP), `job` (background workers) |
| **accessory** | Auxiliary service: databases, Redis, Litestream |
| **volumes** | Local mount directory or Docker volume |
| **Kamal Proxy** | Docker container running on each web server, routes traffic to app containers |

## Install

```bash
gem install kamal
kamal version
```

Or via Dockerized alias (no Ruby required). Latest: `ghcr.io/basecamp/kamal:v2.8.2` or `:latest`.

## The Three Core Commands

### 1. `kamal init` — generate config

```bash
kamal init
# Creates:
#   config/deploy.yml
#   .kamal/secrets
#   .kamal/hooks/ (sample hooks)
```

### 2. `kamal setup` — first-time provision + deploy

Bootstraps servers (installs Docker, creates Kamal network), starts accessories, runs first deploy.

```bash
kamal setup
```

### 3. `kamal deploy` — subsequent deploys

```bash
kamal deploy
```

Builds image → pushes to registry → instructs servers to pull and run → zero-downtime switch.

## Minimal `config/deploy.yml`

```yaml
# config/deploy.yml
service: my-app
image: my-user/my-app

servers:
  web:
    - 192.168.0.1

proxy:
  ssl: true
  host: app.example.com

registry:
  username: my-user
  password:
    - KAMAL_REGISTRY_PASSWORD

builder:
  arch: amd64

env:
  secret:
    - DATABASE_URL
    - RAILS_MASTER_KEY

volumes:
  - "app_storage:/app/storage"

asset_path: /app/public/assets
```

## `.kamal/secrets`

```bash
# Pull secrets from env, 1Password, LastPass, Bitwarden, etc.
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD

# From a file
RAILS_MASTER_KEY=$(cat config/master.key)

# From 1Password CLI
DATABASE_URL=$(op read "op://Vault/item/database_url")
```

## Prerequisites (Before First Deploy)

1. **Dockerfile** — app builds a container with:
   - App server on port `80`
   - `/up` health check route
   - Logs to STDOUT
2. **SSH access** — configured locally and on servers
3. **Docker** — installed on laptop (or CI)
4. **Domain name** — configured with DNS pointing to server
5. **Registry** — Docker Hub, GHCR, DigitalOcean, etc. — to host images

## The Deploy Workflow (What `kamal deploy` Does)

1. Login to Docker registry locally + on servers
2. Build container image from `Dockerfile`, tag with git SHA
3. Push image to registry
4. Ensure Kamal Proxy is running on web servers
5. Check for stale containers
6. Boot app with new version:
   - Extract assets, sync asset volumes
   - Run new container for primary role
   - Health check via Kamal Proxy
   - **Web barrier**: non-primary roles (jobs) wait until primary passes health check
   - Run non-primary role containers
   - Stop old version
   - Clean up assets
   - Tag version as `:latest`
7. Deploy lock file prevents concurrent deploys

## Roles — `web` vs `job`

```yaml
servers:
  web:                        # primary role — serves HTTP, has health check
    - 192.168.0.1
  job:                        # non-primary — background workers
    hosts:
      - 192.168.0.2
    cmd: bin/jobs              # Solid Queue command
```

Only **one role can be primary** (web by default). Primary has health check; non-primary roles (jobs) wait for primary to pass health check.

## Kamal Proxy — Reverse Proxy in Docker

Kamal Proxy runs as a Docker container on each web server. Routes traffic:
- **SSL/TLS via Let's Encrypt** — set `proxy.ssl: true` + `proxy.host: app.example.com`
- **Path-based routing** — for multiple apps
- **Health checks** — `/up` endpoint by default
- **Buffering, forward headers, logging** — configurable

### SSL/TLS (automatic certs)
```yaml
proxy:
  ssl: true
  host: app.example.com    # Let's Encrypt issues cert for this domain
```

Must have DNS pointing domain → server IP before deploy.

## Multi-App on Single VPS (Subdomain Routing)

Key pattern for indie deployments — one VPS, many apps, cheap hosting:

```yaml
# App 1: config/deploy.yml
proxy:
  ssl: true
  host: blog.yourname.com
```

```yaml
# App 2: config/deploy.yml
proxy:
  ssl: true
  host: todo.yourname.com
```

Both point DNS to same server IP. Kamal Proxy routes by `Host:` header. Deploy independently.

**Cost math:** 5 apps on 1 VPS (Rp 75K/month) vs 5 separate VPS (Rp 375K/month) = 80% savings.

## Accessories (Databases, Redis, etc.)

Kamal runs auxiliary services alongside your app:

```yaml
accessories:
  db:
    image: postgres:15
    host: 192.168.0.2
    port: 5432
    env:
      clear:
        POSTGRES_USER: myapp
      secret:
        - POSTGRES_PASSWORD
    directories:
      - data:/var/lib/postgresql/data

  litestream:
    image: litestream/litestream
    host: 192.168.0.1
    cmd: replicate
    volumes:
      - "app_storage:/storage"
    files:
      - config/litestream.yml:/etc/litestream.yml
```

## Secrets — Kamal Env

```yaml
env:
  clear:
    DB_HOST: 192.168.0.2
    RAILS_ENV: production
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
```

Secrets pulled from `.kamal/secrets`. Injected into containers as env vars.

## Hooks — Customize Deploy Flow

Scripts in `.kamal/hooks/`:

| Hook | When |
|---|---|
| `pre-connect` | Before connecting to hosts |
| `pre-build` | Before image is built |
| `pre-deploy` | Before application boot |
| `post-deploy` | After application finished booting |
| `docker-setup` | After Docker install during bootstrap |
| `pre-proxy-reboot` / `post-proxy-reboot` | Around Kamal Proxy reboots |

Populated env vars: `KAMAL_VERSION`, `KAMAL_HOSTS`, `KAMAL_COMMAND`, `KAMAL_ROLE`, etc.

Use for: Slack notifications, DNS warming, custom pre-flight checks.

## Essential Commands

| Command | What it does |
|---|---|
| `kamal init` | Generate config scaffolding |
| `kamal setup` | First-time: bootstrap + deploy |
| `kamal deploy` | Standard deploy (build, push, roll out) |
| `kamal redeploy` | Skip build, re-run existing image |
| `kamal rollback <version>` | Roll back to previous image |
| `kamal app logs` | Tail container logs |
| `kamal app logs -n 100 --grep ERROR` | Filter logs |
| `kamal app exec --interactive --reuse "bash"` | Shell into container |
| `kamal app exec "bin/rails console"` | Run command in container |
| `kamal accessory boot db` | Start accessory |
| `kamal accessory logs db` | Tail accessory logs |
| `kamal proxy logs` | Tail proxy logs |
| `kamal proxy reboot` | Restart Kamal Proxy |
| `kamal prune all` | Clean old images on servers |

## Destinations (Staging / Production)

```yaml
# config/deploy.yml — shared base
# config/deploy.staging.yml — staging overrides
# config/deploy.production.yml — production overrides
```

```bash
kamal deploy -d staging
kamal deploy -d production
```

## Rollbacks

```bash
# List available versions
kamal app details

# Roll back to specific version
kamal rollback abc123

# Same as redeploying previous image
```

**Note:** Rollback only works if previous image still exists on server. Use `kamal prune all` carefully — keeps last 5 by default.

## Debugging

### Logs
```bash
kamal app logs              # live tail
kamal app logs -n 100       # last 100 lines
kamal app logs --grep ERROR # filter
kamal proxy logs            # proxy logs
kamal accessory logs db     # accessory logs
```

### Exec (shell access)
```bash
kamal app exec --interactive --reuse "bash"
kamal app exec "bin/rails runner 'puts User.count'"
```

### Kamal system
```bash
kamal config                # print final config (merged)
kamal lock status           # check deploy lock
kamal lock release          # force unlock
kamal version
```

## Common Failure Modes + Recovery

| Symptom | Likely Cause | Fix |
|---|---|---|
| SSL cert fails on first deploy | DNS not propagated | Wait, `kamal deploy` again |
| Health check times out | App crashing on boot | `kamal app logs`, check for errors |
| "Cannot connect to Docker" | Docker daemon stopped | `sudo systemctl restart docker` |
| Deploy lock stuck | Previous deploy crashed | `kamal lock release` |
| Container keeps restarting | OOM (memory), missing env | Check logs + memory limits |
| Can't push to registry | Registry credentials wrong | Check `.kamal/secrets` |
| Accessory won't boot | Port conflict or config error | `kamal accessory logs <name>` |

## Zero-Downtime Gotchas

- **Asset digest drift** — new deploy removes old assets; if an in-flight request needs old asset, 404. Kamal provides `asset_path:` config to bridge versions temporarily.
- **Database migrations** — run in `pre-deploy` hook or manually before deploy; long migrations can block.
- **Background jobs** — Solid Queue processes from DB queue, safe across deploys. Old Sidekiq jobs may need special handling.

## Anti-Patterns

- ❌ Putting secrets directly in `deploy.yml` (use `env.secret` + `.kamal/secrets`)
- ❌ Running long migrations during `kamal deploy` (blocks deploy)
- ❌ Skipping health check endpoint `/up` (Kamal can't verify boot)
- ❌ Exposing accessory ports publicly (databases should stay internal)
- ❌ `kamal prune` without checking rollback history first
- ❌ Deploying without a git commit (can't rollback precisely)

## References

- Official Kamal docs: https://kamal-deploy.org
- Kamal GitHub: https://github.com/basecamp/kamal
- Josef Strzibny's Kamal Handbook (paid, highly recommended for deep dive)

## Related Skills

- **vps-basics** — minimal VPS prep before Kamal
- **vps-provisioning** — full VPS hardening (Tailscale, Cloudflare) [advanced]
- **rails-debug-helper** — deploy pre-flight checks, log reading
- **solid-suite-config** — Solid Queue runs as Kamal `job` role
