# DHH Vibecoding Plugin

> Vibecoding with the DHH philosophy. Distilled from 37signals' Fizzy, Campfire, and Writebook — production apps shipped with **zero build tools**, **no devise**, **no Sidekiq**, **no Redis**, **no Tailwind**.

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

This plugin ships that philosophy as Claude Code skills. Install once → your Claude Code becomes fluent in vanilla Rails, vanilla CSS, Hotwire, Solid Suite, and Kamal. Your prompts stay light, outputs stay consistent, decisions are curated by "The DHH Way."

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

## 10 Skills Inside

Each skill auto-invokes based on trigger keywords in your prompts.

### Rails Backend
- **`rails-conventions`** — DHH-style patterns (fat models, skinny controllers, REST purity, prefer Rails built-in over gems)
- **`auth-setup`** — Rails 8 built-in auth + authorization + Current attributes (no devise, no pundit)
- **`solid-suite-config`** — Solid Queue / Cache / Cable setup (no Redis, no Sidekiq)
- **`rails-debug-helper`** — error diagnosis, log reading, CI tools (Brakeman, Bundler Audit, Rubocop), deploy pre-flight

### Hotwire (UI)
- **`turbo-frames`** — single-section updates, lazy loading, `turbo_frame_tag` patterns from Fizzy
- **`turbo-streams`** — multi-section updates, broadcasts (declarative + manual), `method: :morph`
- **`stimulus-controllers`** — client-side behavior sprinkles, 55+ Fizzy patterns catalog

### Presentation
- **`vanilla-css`** — cascade layers, OKLCH colors, CSS variables, `:has()`, `@starting-style`, native nesting — zero build tools

### Integrations
- **`mayar-payment-integration`** — Mayar hosted checkout + webhook re-fetch pattern (Indonesian payment gateway)
- **`rubyllm-patterns`** — LLM APIs via RubyLLM gem (Claude/OpenAI/Gemini), streaming, cost-aware model selection

---

## Course Context

This plugin is designed as a companion for the **BelajarGPT Vibecoding** course — teaching non-programmer Indonesians to ship real Rails 8 apps using Claude Code.

The plugin carries the technical load (10 skills auto-invoked). Students focus on **PRD + product thinking**. AI writes the code, student guides + verifies.

Learn more: [BelajarGPT Vibecoding](https://belajargpt.co/vibecoding)

---

## Credits & References

This plugin stands on the shoulders of:
- **DHH** — for the philosophy
- **37signals team** (Jason Fried, Jorge Manrubia, Jason Zimdars, et al.) — for shipping Fizzy, Campfire, Writebook as living proofs
- **Marc Köhlbrugge** — for extracting the patterns in [Unofficial 37signals Coding Style Guide](https://github.com/marckohlbrugge/unofficial-37signals-coding-style-guide)

We synthesized, verified, and re-expressed these patterns as Claude Code skills. Credit flows upstream.

---

## License

MIT — plugin code is yours to use, modify, share.

Reference content adapted from sources above (O'Saasy licensed code examples; please respect their terms).
