# Architecture: Islands Architecture

## Overview

Not a standalone architecture — a hybrid strategy. The majority of the app is server-rendered
(Hotwire, HTMX, or plain Rails), but specific complex interactions are handed off to self-
contained JavaScript "islands" (React, Vue, Svelte, or Lit components).

The term comes from Astro's architecture, but the pattern applies in Rails: most of the page is
static or Turbo-driven HTML, and a `<div id="emoji-picker-root">` gets a React tree mounted
into it. The island owns its own state; the rest of the page doesn't know or care.

## When Islands Make Sense

The honest answer for most features: they don't. Stimulus handles the majority of client-side
interactions cleanly. Islands are justified when:

- The interaction is genuinely complex (rich text editor, virtual scroll, drag-and-drop reorder)
- A mature JS library already exists and reimplementing it in Stimulus would be painful
- The component has significant internal state that doesn't need to sync to the server

Chat-specific candidates:

| Feature | Approach |
|---|---|
| Message list | Turbo Streams — server owns this |
| Send message form | Stimulus or plain HTML |
| Emoji picker | Island — mature libraries exist (emoji-mart), complex state |
| Rich text / markdown editor | Island — Trix (Rails default) or a dedicated editor like TipTap |
| Mention autocomplete (`@user`) | Island or Stimulus, depending on complexity |
| File upload with preview | Island or Stimulus + Active Storage direct upload |
| Reaction picker | Island or Stimulus |
| Message search UI | Turbo Frame is probably sufficient |

## How It Works

```
Page renders via Rails/Turbo as normal.
Specific divs are mount points for JS components.

app/views/channels/show.html.erb:
  <%= turbo_stream_from @channel %>
  <div id="messages"><%= render @channel.messages %></div>

  <%# Island mount point %>
  <div
    id="emoji-picker"
    data-controller="emoji-island"
    data-emoji-island-target-value="#message-body">
  </div>

app/javascript/controllers/emoji_island_controller.js:
  import { Controller } from "@hotwired/stimulus"
  import { createRoot } from "react-dom/client"
  import EmojiPicker from "emoji-mart"

  export default class extends Controller {
    connect() {
      this.root = createRoot(this.element)
      this.root.render(<EmojiPicker onEmojiSelect={this.insert.bind(this)} />)
    }

    disconnect() {
      this.root.unmount()
    }

    insert(emoji) {
      document.querySelector(this.targetValue).value += emoji.native
    }
  }
```

The island is mounted and unmounted by a Stimulus controller — keeping the wiring in the Rails
world while letting the island itself be pure React/Vue.

## Stack

| Layer | Technology |
|---|---|
| Base architecture | Hotwire (or HTMX) — handles routing, real-time, most UI |
| Islands | React, Vue, Svelte, or Lit — only where justified |
| Island wiring | Stimulus controllers mount/unmount island components |
| Build step | Required for the island framework (Vite via `vite_ruby`) |
| Import maps | Still usable for non-island JS; islands need bundling |

## Pros

- **Best of both worlds** — Server rendering and Turbo Streams for real-time; rich JS components
  where needed. You don't sacrifice one for the other.
- **Small JS surface area** — Only the islands are in JS. The rest of the app stays Ruby.
- **Reuse existing libraries** — Drop in TipTap, emoji-mart, or react-beautiful-dnd without
  reimplementing them.
- **Incremental adoption** — Start Hotwire-only. Add an island when you hit a wall. No upfront
  commitment to a full SPA.
- **Islands are isolated** — A bug in the emoji picker doesn't affect message delivery.

## Cons

- **Build step required** — As soon as you add a React/Vue island, you need a JS bundler.
  Import maps alone won't work for JSX or `.vue` files.
- **Two JS worlds** — Stimulus and React coexist; contributors need to know when to reach for
  which. Without clear guidelines this gets messy.
- **Data sharing is awkward** — If an island needs to react to a Turbo Stream update (e.g. "a
  new reaction was added, update the picker state"), bridging the two worlds requires custom
  events or a shared state layer.
- **Bundle size creep** — Each island framework adds weight. React alone is ~45kb gzipped.

## Communication Between Islands and Turbo

Use browser Custom Events as the bridge:

```javascript
// Turbo Stream updates the DOM, Stimulus controller fires a custom event
// Islands listen for it without knowing about Turbo

// In a Stimulus controller (Rails world):
document.dispatchEvent(new CustomEvent("message:created", { detail: { id: 42 } }))

// In a React island:
useEffect(() => {
  const handler = (e) => setLastMessageId(e.detail.id)
  document.addEventListener("message:created", handler)
  return () => document.removeEventListener("message:created", handler)
}, [])
```

## Fit for Hearth

Good incremental strategy. Start with pure Hotwire — it covers 80-90% of the feature list. When
a specific feature proves painful in Stimulus (the emoji picker is the most common example), add
a targeted island rather than migrating the whole app.

This also gives the group a natural conversation arc: start simple, identify friction points,
introduce islands intentionally rather than by default.

## References

- [Vite Ruby](https://vite-ruby.netlify.app) — Vite integration for Rails, needed for island bundling
- [TipTap](https://tiptap.dev) — headless rich text editor, works as a React/Vue island
- [emoji-mart](https://github.com/missive/emoji-mart) — React emoji picker component
- [Astro Islands](https://docs.astro.build/en/concepts/islands/) — where the term originates
- [Stimulus + React mounting pattern](https://dev.37signals.com/haiku-for-stimulus-react/)
