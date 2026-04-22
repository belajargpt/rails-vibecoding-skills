# DHH Vibecoding Plugin

> Vibecoding with the DHH philosophy. Distilled from 37signals' Fizzy, Campfire, and Writebook — production apps shipped with **zero build tools**, **no devise**, **no Sidekiq**, **no Redis**.

*"The best code is the code you don't write. The second best is the code that's obviously correct."* — DHH

---

## Philosophy

Vibecoding needs a north star. We picked **DHH as the vibecoding prophet**.

**Why?**
- **Rich domain models > service objects** — AI doesn't get confused hunting logic across 10 places
- **REST purity** — one way to define endpoints, AI doesn't invent new patterns
- **Convention over configuration** — less decision fatigue, fewer tokens
- **Prefer Rails built-in over gems** — fewer dependencies = fewer failure modes
- **Database-backed everything** — no Redis, no Memcached, Solid Suite ftw

We studied:
- **Fizzy** — 37signals' kanban app (~14K lines of CSS, zero build tools)
- **Campfire** & **Writebook** — shipped proofs of DHH's philosophy
- **Unofficial 37signals Style Guide** — transferable patterns distilled
- **DHH's PR reviews** — real critique patterns from production codebases

This plugin ships that philosophy as Claude Code skills. Install once → your Claude Code becomes fluent in vanilla Rails, Hotwire, Solid Suite, Active Storage, and Kamal. Your prompts stay light, outputs stay consistent, decisions are curated by "The DHH Way."

Your vibecoding partner learns from the best.

---

## Install

```bash
claude plugin marketplace add github:belajargpt/dhh-vibecoding-plugin
claude plugin install dhh-vibecoding
```

## Update

```bash
claude plugin update dhh-vibecoding
```

Quarterly update cycle tracking changes to Rails / Claude Code / Kamal / Mayar.

---

## 16 Skills Inside

Each skill auto-invokes based on trigger keywords in your prompts.

## Main Skills (14)

### Planning
- **`prd-writing`** — 9-section PRD template. The single most important habit for vibecoding — 20 min of PRD saves 2 hours of rework. "Think first, prompt second."

### Rails Backend
- **`rails-conventions`** — DHH-style patterns (fat models, skinny controllers, REST purity, prefer Rails built-in over gems)
- **`auth-setup`** — Rails 8 built-in auth + authorization + Current attributes (no devise, no pundit)
- **`active-storage`** — file uploads, `has_one_attached` / `has_many_attached`, variants, signed URLs for secure delivery
- **`solid-suite-config`** — Solid Queue / Cache / Cable setup (no Redis, no Sidekiq)
- **`rails-debug-helper`** — error diagnosis, log reading, CI tools (Brakeman, Bundler Audit, Rubocop), deploy pre-flight

### Hotwire (UI)
- **`turbo-frames`** — single-section updates, lazy loading, `turbo_frame_tag` patterns from Fizzy
- **`turbo-streams`** — multi-section updates, broadcasts (declarative + manual), `method: :morph`
- **`stimulus-controllers`** — client-side behavior sprinkles, 55+ Fizzy patterns catalog

### Presentation
- **`tailwind-patterns`** — Tailwind CSS 4, mobile-first responsive, accessibility (ARIA, focus states), component patterns (button/card/form/nav). Default styling approach for students.

### Infrastructure
- **`vps-basics`** — minimum viable VPS provisioning (SSH key, UFW, swap, Docker, fail2ban). 15-minute setup for v1 apps.
- **`kamal-deployment`** — Kamal 2 deploy workflow, zero-downtime, deploy.yml config, multi-app subdomain routing, accessories, rollbacks.

### Integrations
- **`mayar-payment-integration`** — Mayar hosted checkout + webhook re-fetch pattern (Indonesian payment gateway)
- **`rubyllm-patterns`** — LLM APIs via RubyLLM gem (Claude/OpenAI/Gemini), streaming, cost-aware model selection

---

## Extras / Advanced (2)

Opt-in skills for DHH-style purists or advanced infrastructure needs. Not auto-invoked unless your prompt explicitly signals the advanced path.

- **`vanilla-css`** — DHH-style vanilla CSS alternative to Tailwind. Cascade layers, OKLCH colors, CSS variables, `:has()`, `@starting-style`, native nesting — zero build tools. For projects that commit to the 37signals styling philosophy.
- **`vps-provisioning`** — advanced VPS hardening with Tailscale private mesh, Cloudflare-only ingress, DOCKER-USER iptables chain, multi-server mesh. Upgrade from `vps-basics` when threat model demands it.

---

## Course Context

This plugin is designed as a companion for the **BelajarGPT Vibecoding** course — teaching non-programmer Indonesians to ship real Rails 8 apps using Claude Code.

The plugin carries the technical load (14 main skills auto-invoked). Students focus on **PRD + product thinking**. AI writes the code, student guides + verifies.

Learn more: [BelajarGPT Vibecoding](https://belajargpt.co/vibecoding)

---

## Credits & References

This plugin stands on the shoulders of:
- **DHH** — for the philosophy
- **37signals team** (Jason Fried, Jorge Manrubia, Jason Zimdars, et al.) — for shipping Fizzy, Campfire, Writebook as living proofs
- **Marc Köhlbrugge** — for extracting the patterns in [Unofficial 37signals Coding Style Guide](https://github.com/marckohlbrugge/unofficial-37signals-coding-style-guide)
- **Josef Strzibny** — for the Kamal Handbook deep dive on Kamal 2
- **Thibaut Baissac** — for Tailwind agent patterns reference

We synthesized, verified, and re-expressed these patterns as Claude Code skills. Credit flows upstream.

---

## License

MIT — plugin code is yours to use, modify, share.

Reference content adapted from sources above (please respect their terms).
