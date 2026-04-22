---
name: turbo-frames
description: Use when building UI with isolated page sections that update independently without full page reload. Applies to inline editing, inline forms, lazy-loaded content sections, dialog/modal content, pagination, filtering UIs, and any "one section swap" pattern. Triggers when user mentions turbo_frame_tag, turbo-frame, inline edit, frame target, lazy load section, frame refresh, morphing frame, target _top, or sectioned update. Turbo Frame = ONE section updates. For broadcast / multi-user updates, see turbo-streams skill. For client-side behavior, see stimulus-controllers skill.
---

# Turbo Frames

> Verified against 37signals Fizzy — 49 `turbo_frame_tag` usages across production code.

## Philosophy

> One frame = one section. Click a link inside the frame → response returns the frame → browser swaps just that frame. **No full page reload.**

## Core Patterns from Fizzy

### Basic frame
```erb
<%= turbo_frame_tag @card, :edit do %>
  <%= link_to "Edit Card", edit_card_path(@card) %>
<% end %>
```

Generates: `<turbo-frame id="card_123_edit">...</turbo-frame>`.

Click link → Turbo fetches the URL → extracts matching `<turbo-frame id="card_123_edit">` from response → swaps in place.

### Composite frame IDs
Fizzy pattern: `turbo_frame_tag @resource, :context_identifier`
```erb
<%= turbo_frame_tag @comment, :container do ... %>     <!-- comment_45_container -->
<%= turbo_frame_tag @comment, :new_reaction do ... %>  <!-- comment_45_new_reaction -->
<%= turbo_frame_tag @card, :watch do ... %>            <!-- card_123_watch -->
<%= turbo_frame_tag @card, :pin do ... %>              <!-- card_123_pin -->
```

Makes multiple frames per resource easy to reference without ID collisions.

### Lazy-loaded frames (`src:` attribute)
```erb
<%= turbo_frame_tag card, :columns, src: edit_card_column_path(card), refresh: "morph" %>
```

Renders empty frame → browser fetches `src` URL on-load → populates frame. Use for:
- Heavy components loaded after main page
- Real-time refresh that needs full fetch
- Sections that depend on external data

### Frame with `refresh: :morph`
```erb
<%= turbo_frame_tag card, :pin, src: card_pin_path(card), refresh: :morph %>
```

When frame is refreshed (e.g., via broadcast), browser uses **morphing** instead of full replacement. Preserves scroll, focus, animations.

### Escape the frame with `target: "_top"`
```erb
<%= turbo_frame_tag card, :columns, src: ..., target: "_top", refresh: "morph" %>
```

Any link/form inside this frame navigates the **whole page** instead of replacing the frame. Useful when click should take user somewhere new.

### Form inside frame (inline edit pattern)
```erb
<%= turbo_frame_tag @card, :edit do %>
  <%= form_with model: @card do |f| %>
    <%= f.text_field :title %>
    <%= f.submit %>
  <% end %>
<% end %>
```

Form submit → controller response includes matching `<turbo-frame id="card_123_edit">` → swap happens.

## When to Use Turbo Frames

✅ **Use Turbo Frame when:**
- Editing one item inline (show → edit form → show again)
- Lazy loading a section (dropdown, tab content, heavy component)
- Pagination (next page swaps the list frame)
- Filter/search UI (form submit updates just the results frame)
- Single-user / single-session update (not broadcast)

❌ **Don't use Turbo Frame when:**
- Update needs to propagate to multiple users/browsers → **use Turbo Streams**
- Multiple sections need to update from one action → **use Turbo Streams**
- Update is client-side only (no server roundtrip) → **use Stimulus**
- You need to append/prepend to a list → **use Turbo Streams**

## Decision Guide: Frame vs Stream vs Stimulus

| Need | Use |
|---|---|
| Swap ONE section on click/submit | Turbo Frame |
| Swap MULTIPLE sections from one action | Turbo Stream |
| Update to all connected users | Turbo Stream + `broadcasts_to` / manual broadcast |
| Append/prepend to list (new comment, notification) | Turbo Stream |
| Show/hide modal, toggle dropdown | Stimulus |
| Auto-resize textarea, debounce input | Stimulus |
| Drag-drop reorder | Stimulus (+ Turbo Frame for server persist) |

## Anti-Patterns

- ❌ Wrapping entire page in one giant frame (defeats the purpose)
- ❌ Using frame for broadcasts (that's streams)
- ❌ Deeply nested frames (can cause confusion with `target: "_top"`)
- ❌ Frame without unique ID (collisions break swap)

## References

- `references/hotwire.md` — comprehensive Hotwire patterns from 37signals Fizzy

## Related Skills

- **turbo-streams** — for multi-section updates, broadcasts, append/prepend patterns
- **stimulus-controllers** — for client-side behavior inside frames
