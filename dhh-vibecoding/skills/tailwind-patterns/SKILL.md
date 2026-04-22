---
name: tailwind-patterns
description: Use when styling Rails ERB views with Tailwind CSS 4 utility classes. Applies to building responsive layouts, mobile-first design, accessibility (ARIA, focus states), color strategy, typography scale, interactive states (hover/focus/active), form styling, card/button/navigation components, and Hotwire-compatible styling. Triggers when user mentions Tailwind, utility classes, responsive design, mobile-first, breakpoint, sm:/md:/lg:/xl:, styling, design, UI layout, form styling, or specific Tailwind utilities. Do NOT trigger for custom CSS (use vanilla-css) or Ruby/JS logic.
---

# Tailwind CSS Patterns — Rails 8 + Hotwire

> Utility-first styling for Rails ERB + ViewComponent. Tailwind CSS 4 with mobile-first responsive design, accessibility built-in, smooth Hotwire integration.

## Philosophy

> Style with utility classes directly in HTML. Readable, grep-able, no CSS file sprawl. Mobile-first by default. Accessibility is not optional.

## Installation (Rails 8)

```bash
bundle add tailwindcss-rails
bin/rails tailwindcss:install
```

Or CDN for quick starter apps (no build step):
```erb
<script src="https://cdn.tailwindcss.com"></script>
```

CDN is fine for v1 / learning. `tailwindcss-rails` gem for production.

## Mobile-First Responsive Design

**Start with mobile styles. Add breakpoints as screen grows.**

| Breakpoint | Min width | Device |
|---|---|---|
| (none) | 0px | Mobile default |
| `sm:` | 640px | Large mobile / small tablet |
| `md:` | 768px | Tablet |
| `lg:` | 1024px | Desktop |
| `xl:` | 1280px | Large desktop |
| `2xl:` | 1536px | Extra large |

### Example
```erb
<div class="flex flex-col gap-4 p-4
            md:flex-row md:gap-6 md:p-6
            lg:gap-8 lg:p-8">
  <div class="w-full md:w-1/3">Sidebar</div>
  <div class="w-full md:w-2/3">Main</div>
</div>
```

**Rule:** Write base styles first (mobile). Add `md:` / `lg:` only where desktop differs.

## Semantic HTML + Accessibility

### Use semantic elements
```erb
<nav class="flex gap-4">...</nav>
<main class="container mx-auto">...</main>
<article class="prose">...</article>
<button type="button" class="btn-primary">...</button>
```

Not `<div>` for everything.

### ARIA attributes
```erb
<%= link_to root_path, aria: { current: request.path == root_path ? "page" : nil } do %>
  Home
<% end %>

<button aria-label="Close dialog" aria-expanded="false">
  <svg class="w-5 h-5">...</svg>
</button>

<span class="sr-only">Screen reader only text</span>
```

### Focus states (keyboard users)
Every interactive element must have visible focus:

```erb
<button class="bg-blue-600 hover:bg-blue-700
               focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2
               active:bg-blue-800
               px-4 py-2 rounded-lg text-white">
  Submit
</button>
```

### Contrast (WCAG AA minimum)
- Text on background: **4.5:1 ratio** for body, **3:1** for large text
- Test with browser dev tools or contrast checker
- Avoid: `text-gray-400` on `bg-white` (too low contrast)
- Prefer: `text-gray-700` on `bg-white`

## Color Strategy (Semantic, Not Decorative)

| Color | Use |
|---|---|
| **Blue** (`blue-600`, `blue-500`) | Primary actions, links |
| **Green** (`green-600`, `green-500`) | Success, confirmations, "safe" |
| **Red** (`red-600`, `red-500`) | Errors, destructive actions |
| **Yellow / Orange** (`yellow-500`, `orange-500`) | Warnings, caution |
| **Gray** (`gray-700`, `gray-500`, `gray-300`) | Neutral text, borders, backgrounds |
| **Indigo / Purple** | Alternative branding, premium |

**Example:**
```erb
<button class="bg-red-600 hover:bg-red-700 text-white">Delete</button>
<button class="bg-blue-600 hover:bg-blue-700 text-white">Save</button>
<p class="text-green-700 bg-green-50 border border-green-200 p-3 rounded">
  Success!
</p>
```

## Typography Scale

| Class | Pixel size | Use |
|---|---|---|
| `text-xs` | 12px | Labels, metadata |
| `text-sm` | 14px | Secondary text |
| `text-base` | 16px | Body text (default) |
| `text-lg` | 18px | Emphasized body |
| `text-xl` | 20px | Small heading |
| `text-2xl` | 24px | H2 |
| `text-3xl` | 30px | H1 |
| `text-4xl` | 36px | Hero h1 |
| `text-5xl` | 48px | Large hero |

Pair with `font-medium` / `font-semibold` / `font-bold` for weight.

## Interactive States (Required for Every Clickable)

```erb
<button class="transition-colors duration-150
               bg-blue-600 text-white px-4 py-2 rounded-lg
               hover:bg-blue-700
               focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2
               active:bg-blue-800
               disabled:bg-gray-300 disabled:cursor-not-allowed">
  Submit
</button>
```

**Must-have states:**
- `hover:` — desktop hover feedback
- `focus:` — keyboard navigation
- `active:` — click feedback
- `disabled:` — when `disabled` attribute is set

### Group hover (parent-child)
```erb
<div class="group">
  <h3 class="text-gray-900">Title</h3>
  <p class="text-gray-500 group-hover:text-gray-700">
    Subtitle darkens when parent hovered
  </p>
</div>
```

## Common Component Patterns

### Button (primary)
```erb
<%= button_tag "Save",
      class: "inline-flex items-center gap-2
              bg-blue-600 hover:bg-blue-700 text-white
              px-4 py-2 rounded-lg font-medium
              focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2
              transition-colors duration-150" %>
```

### Button (secondary)
```erb
<button class="bg-white text-gray-700 border border-gray-300
               hover:bg-gray-50 hover:border-gray-400
               px-4 py-2 rounded-lg font-medium
               focus:outline-none focus:ring-2 focus:ring-gray-400 focus:ring-offset-2">
  Cancel
</button>
```

### Card
```erb
<article class="bg-white rounded-xl shadow-sm border border-gray-200
                p-6 hover:shadow-md transition-shadow">
  <h2 class="text-2xl font-semibold text-gray-900 mb-2"><%= post.title %></h2>
  <p class="text-gray-600 mb-4"><%= post.excerpt %></p>
  <%= link_to "Read more", post,
        class: "text-blue-600 hover:text-blue-800 font-medium" %>
</article>
```

### Form input
```erb
<div class="mb-4">
  <%= f.label :email,
        class: "block text-sm font-medium text-gray-700 mb-1" %>
  <%= f.email_field :email,
        class: "block w-full rounded-lg border-gray-300 shadow-sm
                focus:border-blue-500 focus:ring-blue-500",
        required: true %>
  <% if @user.errors[:email].any? %>
    <p class="mt-1 text-sm text-red-600">
      <%= @user.errors[:email].first %>
    </p>
  <% end %>
</div>
```

### Navigation
```erb
<nav class="bg-white border-b border-gray-200">
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <div class="flex items-center justify-between h-16">
      <%= link_to "Logo", root_path,
            class: "font-bold text-xl text-gray-900" %>

      <div class="hidden md:flex items-center gap-6">
        <% [["Home", root_path], ["About", about_path]].each do |name, path| %>
          <%= link_to name, path,
                aria: { current: request.path == path ? "page" : nil },
                class: "text-gray-700 hover:text-gray-900
                        aria-[current=page]:text-blue-600 aria-[current=page]:font-semibold" %>
        <% end %>
      </div>
    </div>
  </div>
</nav>
```

### Mobile menu (with Stimulus)
```erb
<div data-controller="dropdown" data-action="click@window->dropdown#close">
  <button data-action="click->dropdown#toggle"
          aria-expanded="false"
          aria-label="Toggle menu"
          class="md:hidden p-2 rounded-lg hover:bg-gray-100
                 focus:outline-none focus:ring-2 focus:ring-blue-500">
    <svg class="w-6 h-6">...</svg>
  </button>

  <div data-dropdown-target="menu"
       class="hidden data-[open]:block absolute top-16 inset-x-0
              bg-white border-t border-gray-200 p-4">
    <!-- menu items -->
  </div>
</div>
```

## Hotwire Integration

### Turbo Frame + Tailwind
```erb
<%= turbo_frame_tag @post, :edit, class: "block" do %>
  <%= form_with model: @post do |f| %>
    <%= f.text_field :title,
          class: "block w-full rounded-lg border-gray-300" %>
    <%= f.submit class: "btn-primary" %>
  <% end %>
<% end %>
```

### Stimulus value + Tailwind classes
```erb
<div data-controller="toggle"
     data-toggle-open-class="bg-blue-100"
     data-toggle-closed-class="bg-gray-100"
     class="p-4 rounded-lg transition-colors">
  <!-- initial class from :closed-class, toggles on data-[open] -->
</div>
```

## Testing Across Viewports

Must test at:
- **375px** — mobile (iPhone SE baseline)
- **768px** — tablet (iPad portrait)
- **1024px+** — desktop

Chrome DevTools → Toggle device toolbar → cycle through devices.

Plus: **keyboard navigation** (tab through UI), **screen reader** (VoiceOver on Mac, NVDA on Windows).

## Anti-Patterns

- ❌ `text-gray-400` body text on white (fails WCAG AA contrast)
- ❌ Missing `focus:` states (keyboard users lost)
- ❌ No `disabled:` styling (users confused why button isn't working)
- ❌ Styling `<div>` as button (use `<button>` element)
- ❌ Fixed pixel widths (use responsive: `w-full md:w-64`)
- ❌ Desktop-first styles (always mobile-first)
- ❌ Custom CSS instead of utilities (defeats Tailwind's purpose)
- ❌ `!important` via `!` prefix overuse (indicates bad specificity)
- ❌ Long className strings without grouping (use `class_names` helper)

## When to Reach for Tailwind vs Vanilla CSS

| Scenario | Use |
|---|---|
| Prototyping, quick iteration | Tailwind (speed) |
| Small-medium Rails app | Tailwind or vanilla |
| Design system with many variants | Tailwind (consistency) |
| Marketing pages with unique styles | Either |
| Heavy custom CSS philosophy (DHH-style) | Vanilla (see `vanilla-css` skill) |
| Team prefers readable HTML classes | Vanilla |

Both valid. Pick one and stick to it per project.

## Related Skills

- **vanilla-css** — DHH-style alternative (cascade layers, OKLCH) [extras]
- **turbo-frames** / **turbo-streams** — Hotwire styling patterns
- **stimulus-controllers** — interactive UI behaviors with Tailwind classes
- **rails-conventions** — view structure in Rails MVC
