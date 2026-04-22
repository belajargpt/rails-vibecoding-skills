---
name: turbo-streams
description: Use when pushing updates to browsers from server actions or broadcasting real-time changes to multiple users. Applies to multi-section updates from one request, append/prepend to lists (new comment, new notification), broadcast from model callbacks, Turbo morphing, and multi-user collaboration. Triggers when user mentions turbo_stream, broadcasts_to, broadcast_replace_later_to, real-time, multi-user update, morph, stream action, append/prepend/replace/remove/before/after, format.turbo_stream, or Solid Cable. Turbo Stream = MULTIPLE sections update OR multi-user broadcast. For single-section update, see turbo-frames skill.
---

# Turbo Streams

> Verified against 37signals Fizzy — 17+ `format.turbo_stream` controller actions + extensive manual broadcasts.

## Philosophy

> Server pushes HTML fragments to browsers. Can update multiple DOM targets from one request. Can broadcast to multiple connected users.

## 7 Stream Actions

```ruby
turbo_stream.append     # add to end of target
turbo_stream.prepend    # add to start of target
turbo_stream.before     # insert before target
turbo_stream.after      # insert after target
turbo_stream.replace    # swap target element (outer HTML)
turbo_stream.update     # swap target contents (inner HTML)
turbo_stream.remove     # delete target
```

## Core Patterns from Fizzy

### Multi-action response (.turbo_stream.erb file)
```erb
<!-- app/views/cards/update.turbo_stream.erb -->
<%= turbo_stream.replace dom_id(@card, :card_container),
      partial: "cards/container",
      method: :morph,
      locals: { card: @card.reload } %>

<%= turbo_stream.update dom_id(@card, :edit) do %>
  <!-- new edit form -->
<% end %>
```

One update action → multiple DOM updates. `format.turbo_stream` in controller responds with this template.

### Controller: respond with multiple formats
```ruby
def update
  @card.update!(card_params)

  respond_to do |format|
    format.turbo_stream  # → renders update.turbo_stream.erb
    format.html { redirect_to @card }
  end
end
```

### Morph method (smooth updates)
```erb
<%= turbo_stream.replace dom_id(@card, :card_container),
      partial: "cards/container",
      method: :morph,
      locals: { card: @card.reload } %>
```

`method: :morph` tells Turbo to use morphing algorithm instead of full replacement:
- Preserves input focus
- Preserves scroll position
- Animates changes smoothly
- Better UX for complex updates

### List append (new comment pattern)
```erb
<!-- cards/comments/create.turbo_stream.erb -->
<%= turbo_stream.before [ @card, :new_comment ],
      partial: "cards/comments/comment",
      locals: { comment: @comment } %>

<%= turbo_stream.update [ @card, :new_comment ],
      partial: "cards/comments/new",
      locals: { card: @card } %>
```

Two-step: insert new comment before the form, reset the form.

### Remove (simple deletion)
```erb
<%= turbo_stream.remove @reaction %>
<%= turbo_stream.remove [ @comment, :container ] %>
```

Works with ActiveRecord or `[record, :suffix]` pattern.

## Broadcasts — 2 Patterns

### Pattern A: Declarative macro (`broadcasts_to`)
```ruby
class Item < ApplicationRecord
  belongs_to :list
  broadcasts_to :list
end
```

View subscribes:
```erb
<%= turbo_stream_from @list %>
```

Auto-broadcasts `create` / `update` / `destroy`. Uses default partial (`_item.html.erb`).

**Use when:** standard CRUD, default partial works, no conditional logic.

### Pattern B: Manual `broadcast_*_later_to` (Fizzy style)
```ruby
class Pin < ApplicationRecord
  after_update_commit :broadcast_pin_updates, if: :preview_changed?

  private

  def broadcast_pin_updates
    broadcast_replace_later_to [ user, :pins_tray ],
      partial: "my/pins/pin",
      target: "pin_#{id}"
  end
end

class Notification < ApplicationRecord
  after_create_commit :broadcast_unread
  after_destroy_commit :broadcast_read

  private

  def broadcast_unread
    broadcast_prepend_later_to user, :notifications, target: "notifications"
  end

  def broadcast_read
    broadcast_replace_later_to user, :unread_count
  end
end
```

Methods: `broadcast_append_later_to`, `broadcast_prepend_later_to`, `broadcast_replace_later_to`, `broadcast_remove_to`, `broadcast_update_later_to`.

View subscribes to composite scope:
```erb
<%= turbo_stream_from [current_user, :pins_tray] %>
<%= turbo_stream_from [current_user, :notifications] %>
```

**Use when:**
- Conditional broadcasts (`if: :field_changed?`)
- Multiple broadcast targets per event
- Custom partial or target element
- Need composite scope `[parent, :namespace]`

## When to Use Turbo Streams

✅ **Use Turbo Streams when:**
- Updating multiple DOM sections from one action
- Appending to list (new comment, notification, message)
- Broadcasting to multiple users (real-time collaboration)
- Need `method: :morph` for smooth update UX
- Notification tray, live counters, real-time dashboards

❌ **Don't use Turbo Streams when:**
- Only one section updates from a click/submit → **Turbo Frame simpler**
- Update is client-side only (no server needed) → **Stimulus**
- Simple redirect or page reload works → Rails default redirect

## Decision: When Multiple Sections Update

**Action affects 2+ DOM sections:**
→ Turbo Stream (one `.turbo_stream.erb` file, multiple stream actions)

**Multi-user sees update simultaneously:**
→ Turbo Stream + `broadcasts_to` (declarative) or `broadcast_*_later_to` (manual)

**Input focus / scroll preservation matters:**
→ Turbo Stream with `method: :morph`

## Anti-Patterns

- ❌ Using stream when frame suffices (overkill for single-section update)
- ❌ Broadcasting from synchronous callback (use `_later_to` for async)
- ❌ Broadcasting sensitive data without scoping (permission leak risk)
- ❌ No pagination on broadcast target (performance hit on large lists)
- ❌ `broadcasts_to` on model with complex conditional logic (use manual)

## Security Note

Broadcasts bypass controller authorization. Scope broadcasts correctly:
```ruby
# ❌ BAD — broadcasts to ALL users
Item.broadcasts_to :all

# ✅ GOOD — scoped to workspace members
Item.broadcasts_to ->(item) { item.workspace }
```

View subscription also needs auth check:
```erb
<% if current_user.member_of?(@workspace) %>
  <%= turbo_stream_from @workspace %>
<% end %>
```

## References

- `references/hotwire.md` — comprehensive Hotwire patterns from 37signals Fizzy

## Related Skills

- **turbo-frames** — for single-section updates (simpler than streams)
- **stimulus-controllers** — for client-side behavior
- **solid-suite-config** — for Solid Cable (broadcast backend)
