---
name: vanilla-css
description: "[EXTRAS / ADVANCED] Use when writing CSS for Rails apps without Tailwind, Sass, PostCSS, or any build tools. Applies to CSS architecture, cascade layers, CSS variables, OKLCH colors, dark mode theming, native nesting, :has() selector, @starting-style, container queries, component styling, utility classes, responsive design, and modern CSS features. Triggers when user mentions CSS, styling, vanilla CSS, cascade layers, custom properties, dark mode, OKLCH, Tailwind alternative, native nesting, CSS-in-JS alternative, or responsive design. Philosophy from 37signals (Campfire, Writebook, Fizzy — 14K lines zero build tools). Most students should use tailwind-patterns skill instead; this is for DHH-style purists."
---

# Vanilla CSS — 37signals Style (Extras / Advanced)

> **This is an extras skill.** Most students should use `tailwind-patterns` for v1 apps. Come back here when you want the DHH-style vanilla CSS philosophy — no build tools, pure cascade, modern browser features.

> Patterns verified against 37signals Fizzy production code + article *"Vanilla CSS Is All You Need"* (zolkos.com).

## Philosophy

> CSS has evolved dramatically. The language that once required preprocessors for variables and nesting now has native implementations. 37signals ships Campfire, Writebook, Fizzy — **14,000 lines of CSS, zero build tools.**

**Core principles:**

1. **No build tools.** No Sass, no PostCSS, no Tailwind. Modern CSS does everything.
2. **Semantic component classes** (`btn`, `card`, `dialog`) over utility-first (`flex p-4 rounded-lg`).
3. **CSS variables for everything** — customization via var override, not prop drilling.
4. **Cascade layers** solve specificity wars permanently.
5. **OKLCH for colors** — perceptually uniform, dark mode is one media query.
6. **Utilities as exceptions**, not foundation.

## File Organization (Flat)

One file per concept. No subdirectories. No partials. No import trees.

```
app/assets/stylesheets/
├── _global.css        ← CSS variables (colors, spacing, fonts, easings)
├── reset.css          ← CSS reset
├── base.css           ← body, typography defaults
├── layout.css         ← page-level layout
├── utilities.css      ← one-off helpers (.flex, .gap, .hide)
├── buttons.css        ← .btn component
├── inputs.css         ← form inputs
├── dialog.css         ← modal/dialog
├── cards.css          ← card component
├── [component].css    ← one file per concept
```

*"Zero configuration. Zero build time. Zero waiting."*

## Cascade Layers — Specificity Without Wars

```css
@layer reset, base, components, modules, utilities;

@layer reset {
  *, *::before, *::after { box-sizing: border-box; }
}

@layer base {
  body { font-family: var(--font-sans); }
}

@layer components {
  .btn { /* button styles */ }
  .card { /* card styles */ }
}

@layer utilities {
  .flex { display: flex; }
  .hide { display: none; }
}
```

**Why layers:** Later layers always win, regardless of selector specificity. No more `!important`. File load order doesn't matter.

## CSS Variables — Foundation Pattern

### Spacing System Using `ch` Units
From Fizzy:
```css
:root {
  --inline-space: 1ch;         /* Horizontal — tied to character width */
  --inline-space-half: calc(var(--inline-space) / 2);
  --inline-space-double: calc(var(--inline-space) * 2);

  --block-space: 1rem;          /* Vertical — tied to font size */
  --block-space-half: calc(var(--block-space) / 2);
  --block-space-double: calc(var(--block-space) * 2);
}
```

**Why `ch`:** spacing correlates to content width. Responsive semantically: `@media (min-width: 100ch)` asks *"is there room for 100 characters?"* not *"what device?"*

### Typography Scale
```css
:root {
  --text-x-small: 0.75rem;
  --text-small: 0.85rem;
  --text-normal: 1rem;
  --text-medium: 1.1rem;
  --text-large: 1.5rem;

  @media (max-width: 639px) {
    /* Bump up on mobile for readability */
    --text-small: 0.95rem;
    --text-normal: 1.1rem;
  }
}
```

### Z-Index Scale (Named, Not Magic Numbers)
```css
:root {
  --z-popup: 10;
  --z-nav: 30;
  --z-flash: 40;
  --z-tooltip: 50;
  --z-bar: 60;
  --z-tray: 61;
}
```

### Easing Functions
```css
:root {
  --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-out-overshoot: cubic-bezier(0.25, 1.75, 0.5, 1);
}
```

### Focus Rings (Accessibility Built-In)
```css
:root {
  --focus-ring-color: var(--color-link);
  --focus-ring-offset: 1px;
  --focus-ring-size: 2px;
  --focus-ring: var(--focus-ring-size) solid var(--focus-ring-color);
}

*:focus-visible {
  outline: var(--focus-ring);
  outline-offset: var(--focus-ring-offset);
}
```

## OKLCH Colors — Perceptually Uniform

**Store LCH values as variables, wrap in `oklch()` for colors:**
```css
:root {
  /* LCH components: Lightness Chroma Hue */
  --lch-blue-darkest: 26% 0.126 264;
  --lch-blue-darker: 40% 0.166 262;
  --lch-blue-dark: 57% 0.1895 260;
  --lch-blue-medium: 66% 0.196 257;
  --lch-blue-light: 84% 0.0719 255;
  --lch-blue-lighter: 92% 0.026 254;
  --lch-blue-lightest: 96% 0.016 252;

  /* Named colors use the LCH values */
  --color-link: oklch(var(--lch-blue-dark));
  --color-selected: oklch(var(--lch-blue-lighter));
}
```

**Benefits:**
- **Perceptually uniform** — equal lightness values look equally bright
- **P3 gamut** — wider colors on modern displays
- **Dark mode = flip LCH values** (one media query, no color system rewrite)

### Dark Mode Pattern
```css
/* Light mode defaults in :root above */

html[data-theme="dark"] {
  --lch-ink-darkest: 96% 0.0034 260;  /* Flipped lightness */
  --lch-ink-darker: 86% 0.0061 260;
  /* ... all colors flipped */
}

@media (prefers-color-scheme: dark) {
  html:not([data-theme]) {
    /* Same flipped values for system pref when no manual override */
  }
}
```

### Semantic Color Names
Don't hardcode colors in components. Name by meaning:
```css
:root {
  --color-canvas: oklch(var(--lch-white));
  --color-ink: oklch(var(--lch-ink-darkest));
  --color-link: oklch(var(--lch-blue-dark));
  --color-negative: oklch(var(--lch-red-dark));
  --color-positive: oklch(var(--lch-green-dark));
  --color-selected: oklch(var(--lch-blue-lighter));
  --color-highlight: oklch(var(--lch-yellow-lighter));
}
```

## Component Pattern — Variables All the Way Down

From Fizzy `buttons.css`:
```css
@layer components {
  .btn {
    /* Component-level variables (overridable) */
    --btn-border-radius: 99rem;
    --btn-hover-brightness: 0.9;

    align-items: center;
    background-color: var(--btn-background, var(--color-canvas));
    border-radius: var(--btn-border-radius);
    border: var(--btn-border-size, 1px) solid var(--btn-border-color, var(--color-ink-light));
    color: var(--btn-color, var(--color-ink));
    padding: var(--btn-padding, 0.5em 1.1em);

    @media (any-hover: hover) {
      &:hover {
        filter: brightness(var(--btn-hover-brightness));
      }
    }
  }

  /* Variants just override variables */
  .btn--negative {
    --btn-background: var(--color-negative);
    --btn-color: var(--color-ink-inverted);
  }

  .btn--circle {
    --btn-border-radius: 50%;
    --btn-padding: 0.75em;
  }
}
```

**HTML stays readable:**
```html
<button class="btn btn--negative">Delete</button>
<button class="btn btn--circle">+</button>
```

**Callers can override variables:**
```css
.my-section .btn {
  --btn-border-radius: 4px;  /* Section-specific buttons */
}
```

## Native Nesting (No Preprocessor)

```css
.card {
  padding: var(--block-space);

  &:hover {
    background: var(--color-selected-light);
  }

  .card__title {
    font-size: var(--text-large);
  }

  @media (max-width: 639px) {
    padding: var(--block-space-half);
  }
}
```

Works in all modern browsers. No Sass needed.

## `:has()` Selector — Parent Selection

State-based styling previously needing JavaScript:
```css
/* Style card differently when it has a complete step */
.card:has(.step--complete) {
  opacity: 0.6;
}

/* Form input + label pattern */
.field:has(input:focus) .label {
  color: var(--color-link);
}

/* Sidebar has mobile menu open */
html:has(.menu[data-open]) {
  overflow: hidden;
}
```

## `@starting-style` — Dialog Entrance Animation

From Fizzy `dialog.css`:
```css
@layer components {
  .dialog {
    opacity: 0;
    transform: scale(0.2);
    transition: var(--dialog-duration) allow-discrete;
    transition-property: display, opacity, overlay, transform;

    &[open] {
      opacity: 1;
      transform: scale(1);
    }

    @starting-style {
      &[open] {
        opacity: 0;
        transform: scale(0.2);
      }
    }
  }
}
```

`@starting-style` gives browsers the "before" state for smooth CSS-only dialog open/close. No JavaScript animation libraries.

## Container Queries — Component Responsive

```css
.card {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 400px) {
  .card__header { display: flex; }
}
```

Component responsive without viewport breakpoints — card adapts to its container, not the window.

## Responsive Sizing — clamp() > Media Queries

```css
:root {
  --main-padding: clamp(var(--inline-space), 3vw, calc(var(--inline-space) * 3));
  --tray-size: clamp(12rem, 25dvw, 24rem);
  --text-fluid: clamp(1rem, 2vw, 1.5rem);
}
```

Size scales with viewport smoothly, no breakpoint cliffs. `dvw`/`dvh` = dynamic viewport (handles mobile URL bar).

## Utilities — Exceptions, Not Foundation

From Fizzy `utilities.css`:
```css
@layer utilities {
  .flex { display: flex; }
  .gap { gap: var(--inline-space); }
  .hide { display: none; }
  .txt-center { text-align: center; }
  .txt-small { font-size: var(--text-small); }
  .txt-subtle { color: var(--color-ink-dark); }
}
```

**Use sparingly** — one-off adjustments, not entire component styling. Inverts Tailwind's philosophy.

## Naming Conventions

| Pattern | Example |
|---|---|
| Semantic component | `.btn`, `.card`, `.dialog` |
| BEM-like variants | `.btn--negative`, `.btn--circle` |
| BEM-like elements | `.card__header`, `.card__footer` |
| Descriptive variables | `--lch-blue`, `--color-link`, `--btn-padding` |
| Utilities — short + clear | `.flex`, `.gap`, `.pad`, `.hide` |
| Text utilities | `.txt-small`, `.txt-subtle`, `.txt-nowrap` |
| Accessibility | `.for-screen-reader`, `.visually-hidden` |

## When to Use Vanilla CSS

✅ **Use vanilla CSS when:**
- Building Rails app with Turbo/Hotwire
- Want zero build tools (instant feedback in dev)
- Care about dark mode done right (OKLCH)
- Value semantic HTML/CSS readability
- Small-to-medium app (< 20K lines CSS)
- Team includes designers reading CSS directly

❌ **Don't use vanilla CSS when:**
- Design system shipped across 20+ projects (design tokens infra makes sense)
- Team conventionally prefers utility-first (Tailwind) — valid preference
- Need CSS-in-JS for component libraries shipping to other apps
- Legacy codebase deeply invested in Sass/PostCSS

## Decision: Vanilla vs Tailwind

| Factor | Vanilla CSS | Tailwind |
|---|---|---|
| Build tools | None | Required |
| HTML readability | Clean classes | Verbose |
| Learning curve | CSS fundamentals | Utility memorization |
| Customization | CSS variables | Theme config + plugins |
| Dark mode | OKLCH flip | Requires config |
| Specificity | Cascade layers solve it | Utility win via order |
| Shared components | Custom CSS files | Utility repetition or @apply |

Neither is "wrong." Match team preference + project scale.

## Anti-Patterns

- ❌ Using `!important` (cascade layers make this unnecessary)
- ❌ Hardcoded colors in components (use semantic CSS variables)
- ❌ Deep nesting (2-3 levels max for readability)
- ❌ Magic z-index numbers (use named scale)
- ❌ Fixed pixel sizes everywhere (use `rem`, `ch`, `dvw`)
- ❌ Duplicating colors across files (centralize in `_global.css`)
- ❌ Over-utilities (if you're writing `<div class="flex p-4 gap-2 items-center">` everywhere, make a component)

## UI Affordances (Pure CSS, No JS)

- **Dialog entrance animations** → `@starting-style` + `allow-discrete`
- **Loading spinners** → CSS masks + `@keyframes`
- **Highlighter effect on `<mark>`** → asymmetric `border-radius` pseudo-elements + `mix-blend-mode`
- **Parent-state styling** → `:has()` selector
- **Responsive sizing** → `clamp()`, `min()`, `max()`
- **Focus management** → `:focus-visible` + named focus ring variables

## References

- `references/css.md` — comprehensive 37signals CSS architecture guide (cascade layers, OKLCH, dark mode, modern features)

## Related Skills

- **rails-conventions** — CSS fits into overall DHH-style Rails app (no Sass, no bundlers)
- **turbo-frames** — frame-scoped styling patterns
- **stimulus-controllers** — for CSS class toggling via JS
