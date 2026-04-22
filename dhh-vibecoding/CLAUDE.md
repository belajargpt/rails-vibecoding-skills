# DHH Vibecoding — Project Conventions

> *"The best code is the code you don't write. The second best is the code that's obviously correct."* — DHH

This plugin teaches Claude Code to write Rails apps **the DHH way** — distilled from 37signals production apps (Fizzy, Campfire, Writebook). The context below loads every session so code generation, debugging, and deployment stay consistent with this philosophy.

## Core Principle

**DHH is our vibecoding prophet.** Why?

1. **Rich domain models > service objects** → AI doesn't get confused hunting for logic
2. **REST purity** → one way to define endpoints, AI doesn't invent new patterns
3. **Convention over configuration** → fewer decisions, fewer tokens
4. **Prefer Rails built-in** → fewer dependencies = fewer failure modes
5. **Database-backed everything** → no Redis, no Memcached, no Sidekiq

This isn't dogma — it's proof. 37signals ships Fizzy (kanban), Campfire (chat), Writebook (publishing) on this stack. ~14K lines of CSS, zero build tools. Built by small teams, maintained by humans, AI-friendly by design.

## Stack (Non-Negotiable)

| Layer | Choice | DHH Rationale |
|---|---|---|
| Framework | **Rails 8.1** | Majestic monolith, opinionated |
| Frontend | **Hotwire (Turbo + Stimulus)** + importmap | No JS toolchain, HTML as wire protocol |
| CSS | **Vanilla CSS** (no Sass, no Tailwind) | Modern CSS is plenty (layers, OKLCH, `:has()`) |
| Database | **SQLite + Litestream** | Zero DB ops, file-based |
| Queue / Cache / Cable | **Solid Queue / Cache / Cable** | Database-backed, no Redis |
| Deployment | **Kamal 2** → VPS | Own infrastructure, Docker-based |
| Payment | **Mayar** (Indonesia-native) | Hosted checkout, webhook re-fetch pattern |
| Cloud storage | **Local filesystem** (v1) / R2 optional | Simplicity first |
| LLM integration | **RubyLLM gem** | Provider-agnostic, streaming, cost-aware |
| Primary AI tool | **Claude Code** | The developer's coding partner |

## What DHH (and This Plugin) Deliberately Avoids

- ❌ **devise** — Rails 8 built-in auth (~150 lines) is enough
- ❌ **pundit / cancancan** — simple role checks in models
- ❌ **sidekiq** — Solid Queue uses the database
- ❌ **redis** — database for everything
- ❌ **view_component** — partials work fine
- ❌ **GraphQL** — REST with Turbo is sufficient
- ❌ **factory_bot** — fixtures are simpler
- ❌ **rspec** — Minitest ships with Rails
- ❌ **Tailwind CSS** — vanilla CSS with layers + OKLCH
- ❌ **Sass / PostCSS** — modern native CSS replaces them

When generating Rails code, default to these omissions. Don't suggest them as alternatives.

## Core Vibecoding Principles

1. **PRD before code.** The user is a product owner, not a programmer. Plan first, prompt second.

2. **One way to do things.** Don't offer 3 alternatives — pick the DHH default and implement it. Users aren't equipped to choose between patterns.

3. **Explain via analogy, not jargon.** "Session = a temporary ID card the server remembers" beats "session cookie with signed token."

4. **Prompt-driven workflow, not custom DSL.** User prompts *"commit with a clear message + push"* — you handle git via Bash. No custom slash commands.

5. **Use Kamal directly.** `kamal setup`, `kamal deploy`, `kamal rollback`. Don't wrap in custom commands.

6. **Mobile-first.** The target audience uses mobile 70%+ of the time. Test in mobile viewport before declaring done.

7. **No SMTP dependency.** Password reset → admin panel. Digital delivery → on-screen signed URL.

8. **Active Storage = local filesystem.** VPS disk is sufficient for v1. Mention R2 as an upgrade path, don't configure by default.

## User Context

- **Primary audience:** Mid-career Indonesian professional, 35+ (non-programmer)
- **Background:** Marketer, founder, creator, consultant, small business owner
- **Goal:** Ship real web apps (not become a senior dev)
- **Language:** Indonesian primary, English technical terms preserved
- **Environment:** Mac-first, Windows via WSL2

## Common Workflows

### Save point (every session)
User: *"commit changes with a clear message + push to GitHub"*
→ You: read the diff, generate a descriptive commit message, `git add` + `git commit` + `git push`

### Deploy (from M3 onwards)
User: *"check if the app is ready to deploy + deploy"*
→ You: run `bin/rails test`, validate Kamal config + SSH + DNS + Docker, THEN `kamal deploy`. Use `kamal rollback` if broken.

### Error debugging
User pastes error verbatim + context.
→ You: read error, check `log/development.log` or `kamal app logs`, diagnose, propose fix.

### Multi-app VPS (M4 onwards)
User has 1 VPS hosting N apps via Kamal proxy + subdomains.
→ When deploying the Nth app, update `deploy.yml` with subdomain host, add DNS A record, keep existing apps running.

## The Skill Catalog

11 skills, each auto-invokes when a prompt matches its trigger keywords:

**Rails backend:**
- `rails-conventions` — fat models, skinny controllers, REST purity
- `auth-setup` — Rails 8 auth, Current attributes, permission isolation
- `solid-suite-config` — Queue / Cache / Cable (Redis-free)
- `rails-debug-helper` — error diagnosis, CI tools, pre-flight

**Hotwire:**
- `turbo-frames` — single-section updates
- `turbo-streams` — multi-section + broadcasts
- `stimulus-controllers` — client-side behavior

**Presentation:**
- `vanilla-css` — cascade layers, OKLCH, no Tailwind

**Infrastructure:**
- `vps-provisioning` — SSH lockdown, Tailscale, UFW, Docker port discipline, Cloudflare-only iptables

**Integrations:**
- `mayar-payment-integration` — Mayar + webhook re-fetch
- `rubyllm-patterns` — LLM APIs via RubyLLM gem

Each skill has detailed references from 37signals' Fizzy codebase + verified official docs.

## Reference Sources

- [Unofficial 37signals Coding Style Guide](https://github.com/marckohlbrugge/unofficial-37signals-coding-style-guide) — transferable patterns
- [37signals Fizzy production code](https://github.com/basecamp/fizzy) — living proof of DHH's style
- [Rails Guides](https://guides.rubyonrails.org) — for Rails 8 specific features
- [RubyLLM docs](https://rubyllm.com) — for LLM patterns
- [Mayar docs](https://docs.mayar.id) — for payment integration
