---
name: stimulus-controllers
description: Use when adding client-side behavior to HTML without a full JS framework. Applies to dialog/modal toggle, dropdown, hotkey handling, form auto-submit, auto-resize textarea, drag-and-drop, copy-to-clipboard, lightbox, tooltip, theme switcher, timezone cookie, combobox, or any "sprinkle of interactivity" pattern. Triggers when user mentions Stimulus, data-controller, data-action, data-target, client-side behavior, JS without framework, show/hide, dropdown, modal, hotkey, auto-submit, or interactive UI element. For server-driven updates, see turbo-frames or turbo-streams skills.
---

# Stimulus Controllers

> Verified against 37signals Fizzy — 55+ Stimulus controllers in production.

## Philosophy

> Stimulus sprinkles behavior onto HTML. Not a framework. Not a view layer. Behavior lives in HTML via `data-*` attributes.

## Anatomy of a Stimulus Controller

```html
<div data-controller="dialog"
     data-action="keydown.esc->dialog#close click@document->dialog#closeOnClickOutside"
     data-dialog-open-class="dialog--open"
     data-dialog-url-value="/cards/new">

  <button data-action="click->dialog#toggle">Open</button>

  <div data-dialog-target="modal">...</div>
</div>
```

```js
// app/javascript/controllers/dialog_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "modal" ]
  static classes = [ "open" ]
  static values = { url: String }

  toggle() {
    this.modalTarget.classList.toggle(this.openClass)
  }

  close(event) {
    if (event.key === "Escape") {
      this.modalTarget.classList.remove(this.openClass)
    }
  }

  closeOnClickOutside(event) {
    if (!this.element.contains(event.target)) {
      this.modalTarget.classList.remove(this.openClass)
    }
  }
}
```

## Static Properties

| Property | Purpose | Example |
|---|---|---|
| `targets` | DOM elements inside controller scope | `static targets = ["modal", "button"]` |
| `classes` | CSS classes used programmatically | `static classes = ["open", "disabled"]` |
| `values` | Data passed from HTML (typed) | `static values = { url: String, count: Number }` |
| `outlets` | References to other controllers | `static outlets = ["tooltip"]` |

Access in methods:
- `this.modalTarget`, `this.modalTargets` (array)
- `this.openClass` (string value)
- `this.urlValue` (typed value from `data-dialog-url-value`)

## Action Syntax

```html
<!-- Basic action -->
<button data-action="click->my-controller#method">

<!-- Global event (window/document) -->
<button data-action="keydown.esc@window->my-controller#close">
<button data-action="click@document->my-controller#outsideClick">

<!-- Multiple actions on one element -->
<button data-action="click->alpha#foo mouseover->beta#bar">

<!-- Event parameter modifiers -->
<button data-action="keydown.ctrl+s->my-controller#save">
```

## Lifecycle Callbacks

```js
export default class extends Controller {
  initialize() { /* once, before first connect */ }
  connect()    { /* each time element enters DOM */ }
  disconnect() { /* each time element leaves DOM */ }
  xTargetConnected()    { /* when target appears */ }
  xTargetDisconnected() { /* when target removed */ }
  xValueChanged()       { /* when value changes */ }
}
```

Use `connect()` for setup, `disconnect()` for cleanup (teardown event listeners, timers).

## Fizzy Controller Catalog (Reusable Patterns)

55+ production controllers covering common needs. Top reusable ones:

### UI Interactions
- `dialog_controller.js` — modal open/close with ESC + outside click
- `dropdown_controller.js` — show/hide menus
- `tooltip_controller.js` — hover tooltips
- `lightbox_controller.js` — image lightbox

### Form Helpers
- `auto_save_controller.js` — auto-save on input change
- `auto_submit_controller.js` — submit form on select change
- `autoresize_controller.js` — textarea grows with content
- `form_controller.js` — custom form submission
- `combobox_controller.js` — search + select dropdown
- `multi_selection_combobox_controller.js` — multi-select combobox

### Navigation
- `hotkey_controller.js` — keyboard shortcuts (Cmd+K, ESC, etc.)
- `navigable_list_controller.js` — arrow key navigation in lists

### Advanced UI
- `drag_and_drop_controller.js` — drag-drop reorder (HTML5 drag API)
- `drag_and_strum_controller.js` — musical drag interactions
- `copy_to_clipboard_controller.js` — copy text to clipboard
- `syntax_highlight_controller.js` — code highlighting

### System / Meta
- `theme_controller.js` — light/dark theme switcher
- `timezone_cookie_controller.js` — set user timezone cookie
- `turbo_navigation_controller.js` — Turbo navigation helpers
- `element_removal_controller.js` — remove element with animation

## When to Use Stimulus

✅ **Use Stimulus when:**
- Show/hide UI elements (dropdown, modal, accordion)
- Handle keyboard shortcuts / hotkeys
- Form helpers (auto-save, auto-resize, debounce)
- Drag-and-drop interactions
- Copy to clipboard, syntax highlighting, tooltips
- Theme switching, timezone detection
- Progressive enhancement (app works without JS, better with)

❌ **Don't use Stimulus when:**
- Update needs server data → **Turbo Frame or Stream**
- Real-time broadcast to multiple users → **Turbo Stream**
- Heavy stateful UI → might need React/Vue (edge case, Rails rarely needs)
- One-line DOM manipulation inline (just use `onclick`)

## Decision: Stimulus vs Turbo

| Need | Use |
|---|---|
| Toggle modal visibility | Stimulus |
| Submit form and update UI from server | Turbo Frame |
| Appearance change (theme, color) | Stimulus |
| Server-persisted state change | Turbo Frame/Stream |
| Keyboard shortcut that navigates | Stimulus (or `data-turbo-action`) |
| Drag-drop UI + server persist | Stimulus (drag) + Turbo Frame (save) |
| Live counter from server | Turbo Stream broadcast |
| Debounce search input | Stimulus (then Turbo Frame update) |

## Anti-Patterns

- ❌ Stimulus controller that fetches HTML and swaps it (that's Turbo's job)
- ❌ Huge controller with 500+ lines (split into smaller controllers)
- ❌ Controllers depending on jQuery (use plain JS)
- ❌ No `disconnect()` cleanup when adding event listeners (memory leak)
- ❌ Stimulus for everything (Turbo handles most server interactions)

## Controller Best Practices

1. **One responsibility per controller** (dialog, dropdown, tooltip — separate)
2. **Private methods prefixed with `#`** (native JS private syntax)
3. **Static properties declared first** (targets, classes, values)
4. **Lifecycle first, public actions second, private methods last**
5. **Use `values` for config** (better than data-attributes directly)
6. **Outlet API for inter-controller communication** (cleaner than events)

## References

- `references/stimulus.md` — reusable controller catalog with copy-paste code (from 37signals)

## Related Skills

- **turbo-frames** — for server-driven section updates
- **turbo-streams** — for multi-section + real-time broadcasts
