---
name: prd-writing
description: Use when planning a new feature, app, or product before writing any code. Applies when user wants to write a PRD (Product Requirements Document), spec out a feature, capture requirements, scope a project, define MVP, or translate an idea into something Claude Code can build. Triggers when user mentions PRD, requirements, spec, plan before code, "think first prompt second", scope, must-have, out-of-scope, user journey, success criteria, or feature planning. The single most important habit for vibecoding — 20 min of PRD saves 2 hours of rework.
---

# PRD Writing — Think First, Prompt Second

## Philosophy

> The biggest mistake non-programmers make with AI coding: prompt immediately without planning. Result is wrong every time.

A Product Requirements Document (PRD) is a short, structured spec you write **before** prompting Claude Code. It becomes the context every prompt references:

```
"Berdasarkan PRD saya [paste], bikin [feature]..."
```

Without a PRD, Claude Code guesses. With a PRD, Claude Code aligns.

## The 9-Section PRD Template

Not "lite" — full. Shallow PRDs lead to scope creep and misalignment. Commit to 20 minutes, get a thousand-fold return.

| # | Section | What to Capture |
|---|---|---|
| 1 | **Problem** | One paragraph: what pain are you solving? Who feels it? |
| 2 | **Solusi singkat** | One-sentence pitch: what this app does |
| 3 | **Target user** | Concrete persona. Age, role, context. |
| 4 | **User journey** | Step-by-step: how a user goes from first visit to successful outcome |
| 5 | **Must-have features (v1)** | 5-10 features. Everything else is v2. |
| 6 | **Nice-to-have (v2)** | Features that can wait |
| 7 | **Out-of-scope (v1)** | **5+ items explicitly NOT building.** Discipline against scope creep. |
| 8 | **Success criteria** | Measurable outcomes (not "it works" — "10 customers, X NPS, Y uptime") |
| 9 | **Mockup / flow sketch** | Paper sketch or draw.io. Even simple boxes count. |

## Where to Store It

```
your-project/
└── docs/
    └── PRD.md          ← commit to git, reference in every prompt
```

## Process

1. **Draft** — fill in all 9 sections. Takes 20-30 min for v1 apps, up to 1 hour for complex ones.
2. **Review via AI** — prompt: *"Review PRD ini, identify ambiguity di scope atau success criteria."*
3. **Revise** — tighten based on feedback.
4. **Commit to git** — PRD is now source of truth.
5. **Reference in prompts** — *"Berdasarkan PRD saya [paste relevant sections], implement X."*

## PRD Variants by Use Case

PRDs adapt to context. Same 9-section skeleton, different emphasis:

### App PRD (e.g., Blog, Todo, Social App)
- Emphasis on **user journey** + **must-have features**
- Out-of-scope is often "auth variations, exotic integrations, v2 UX polish"

### Product PRD (e.g., Digital Product Sales Page)
- Emphasis on **target buyer persona** + **pricing / positioning** (add as subsection under Target user)
- Must-have includes conversion elements: sales copy, social proof, CTA
- Success criteria: conversion rate, revenue target

### AI Feature PRD
- Emphasis on **AI scope boundary**: what AI handles, what stays manual
- Must-have includes **cost budget** (monthly AI API spend)
- Success criteria: classification accuracy, response time, cost/user

### Multi-User App PRD
- Emphasis on **permission boundaries**: who sees what, who can edit what
- Out-of-scope often includes: complex RBAC, multi-tenant isolation
- Success criteria: zero permission leaks, N collaborators working without conflict

## Example — Blog PRD

```markdown
## Problem
I want my own blog — a place I own, not a platform that can ban me or change its algorithm.

## Solusi singkat
Personal Rails blog with rich-text posts, categories, comments, hero images. Self-hosted on my VPS.

## Target user
Me (the author, 30-40 year old professional) + casual readers landing from social media.

## User journey
1. Reader lands from TikTok/IG link → reads post → shares via copy link
2. Author (me) opens admin → writes new post with ActionText → publishes
3. Commenter reads post → drops comment (no account needed, email + name only)

## Must-have features (v1)
- CRUD posts with rich text
- Categories + tags
- Public comments (anon OK)
- Hero image per post
- Mobile-first design
- SEO-friendly URLs

## Nice-to-have (v2)
- Newsletter signup
- Related posts
- Comment moderation

## Out-of-scope (v1)
- Multi-author support
- Paid subscriptions / paywall
- Complex analytics
- Image gallery / multi-image posts
- Social login for commenters

## Success criteria
- Publish 10+ posts in first month
- 99% uptime
- 3+ public comments from real readers
- Mobile Lighthouse score > 90

## Mockup / flow
[paper sketch: homepage → post page → comment form]
```

## Why Full (not Lite) Matters

**Out-of-scope section** is where PRD earns its keep:
- "We're NOT building X" prevents scope creep mid-build
- Forces discipline: decide now what's important
- Student says yes to the few, not everything

**Success criteria** prevents infinite tweaking:
- Without metrics, student polishes forever
- With "10 customers in 2 weeks" or "99% uptime," project has finish line

**Mockup** catches gaps text misses:
- Drawing forces you to think about the user flow spatially
- Reveals missing screens, unclear transitions

## Anti-Patterns

- ❌ Starting to prompt without a PRD ("let's just see what happens")
- ❌ Skipping out-of-scope section (everything ends up must-have)
- ❌ Vague success criteria ("it should work well")
- ❌ PRD stored in head, not committed to git (falls out of sync)
- ❌ Writing PRD after code (reverse engineering, loses planning value)
- ❌ Over-engineering PRD (20-30 min, not 2 days)

## How to Use PRD in Prompts

**Bad:**
> "Build me a blog"

**Good:**
> "Berdasarkan PRD di docs/PRD.md [paste], scaffold the Post model with all must-have fields (title, body ActionText, category, hero_image via Active Storage, published_at). Make sure you respect the out-of-scope section."

**Better (iterative):**
> Use PRD as conversational anchor throughout project. Reference specific sections:
> - "Check if this implementation satisfies must-have #3 and #5"
> - "This feature drifts into v2 scope per PRD section 6, let's defer it"

## When to Skip PRD

Rarely. But acceptable shortcuts:
- **1-file prototype** that you'll throw away
- **Extending existing app** with a feature already scoped
- **Debug / refactor** tasks (PRD was for initial build)

For anything you'll ship to users — always PRD first.

## Related Skills

- **rails-conventions** — after PRD, Claude Code implements DHH-style
- **vanilla-css** / **tailwind-patterns** — UI implementation after PRD
- **auth-setup** — when PRD says "multi-user" or "login required"
