# Architecture: Alpine.js + HTMX

## Overview

A hypermedia-driven approach that keeps logic on the server but uses different client-side
libraries than the Rails-default Hotwire stack. HTMX handles AJAX requests and HTML swapping —
the server returns HTML fragments, HTMX drops them into the DOM. Alpine.js handles local
client-side state and interactivity directly in markup.

Conceptually similar to Hotwire (server renders HTML, client swaps it in) but framework-agnostic
— no Rails-specific magic, just HTTP and HTML.

## How It Works

```
User submits a message:
  HTMX → POST /channels/42/messages  (hx-post attribute on the form)
        ← HTML fragment (rendered message partial)
  HTMX swaps the fragment into #messages

Real-time updates:
  HTMX SSE → GET /channels/42/stream  (hx-ext="sse", sse-connect="/channels/42/stream")
            ← text/event-stream (Rails ActionController::Live or SSE adapter)
  HTMX listens for named events and swaps HTML into the target
```

HTMX attributes live in the HTML:
```html
<!-- Subscribe to SSE stream, append new messages -->
<div id="messages"
     hx-ext="sse"
     sse-connect="/channels/42/stream"
     sse-swap="new_message"
     hx-swap="beforeend">
  <%= render @messages %>
</div>

<!-- Submit form without page reload -->
<form hx-post="/channels/42/messages" hx-swap="none">
  <input type="text" name="body" />
  <button type="submit">Send</button>
</form>
```

Alpine.js handles local state — things that don't need a server round-trip:

```html
<!-- Emoji picker toggle -->
<div x-data="{ open: false }">
  <button @click="open = !open">😊</button>
  <div x-show="open">...picker...</div>
</div>
```

## Stack

| Layer | Technology |
|-------|-----------|
| AJAX + HTML swapping | [HTMX](https://htmx.org) |
| Client-side interactivity | [Alpine.js](https://alpinejs.dev) |
| Real-time transport | SSE (`ActionController::Live`) or Action Cable |
| Server | Rails (renders HTML partials, streams SSE) |
| Asset pipeline | Import maps (both libs available via CDN/importmap) |
| No build step needed | Both libraries are small enough to ship via import maps |

## Pros

- **No build step** — Both HTMX and Alpine ship as single JS files, compatible with import maps.
- **Framework-agnostic** — HTMX works with any backend; knowledge transfers outside Rails.
- **Hypermedia philosophy** — Server is the source of truth for all UI state. Very simple mental
  model: the server returns HTML, the browser shows it.
- **SSE is simpler than WebSockets** — Server-sent events are one-directional and stateless;
  easier to reason about and debug than a persistent WebSocket connection.
- **Alpine is lighter than React/Vue** — ~15kb. Declarative interactivity without a virtual DOM.
- **Progressive enhancement** — HTMX degrades gracefully; pages work without JS for simple cases.

## Cons

- **HTMX SSE vs Action Cable** — HTMX's SSE extension is simpler but less capable than Action
  Cable. Broadcast to specific users/rooms requires careful SSE stream management in Rails
  (`ActionController::Live` keeps a thread alive per connected client — can be costly at scale).
- **Action Cable is an option but awkward** — HTMX can connect to Action Cable WebSockets via
  the `ws` extension, but it's less ergonomic than Turbo Streams.
- **Alpine has limits** — Works well for local UI state (dropdowns, toggles, form behavior) but
  gets unwieldy for complex shared state across components.
- **Less Rails convention** — No `broadcast_append_to` helpers; you write SSE endpoints manually.
- **Smaller Rails community** — Fewer tutorials, gems, and patterns compared to Hotwire.

## SSE Threading Concern

`ActionController::Live` keeps a thread open per connected client. With Puma's default thread
pool, this limits concurrent connections. Mitigations:

- Increase Puma threads (`config/puma.rb`)
- Use an async adapter (Falcon web server)
- Fall back to Action Cable for the real-time layer (HTMX `ws` extension)

## Fit for Hearth

Interesting middle path. The hypermedia model aligns with Rails values, and the no-build-step
story is as clean as Hotwire. The main friction is real-time: SSE works but requires managing
long-lived connections, and Action Cable integration with HTMX is less polished than Turbo
Streams.

A good option if the group wants to explore outside the Rails default ecosystem while staying
server-rendered. Also a natural conversation topic for a Ruby group — the hypermedia/REST
philosophy behind HTMX is worth discussing on its own merits.

## References

- [HTMX docs](https://htmx.org/docs/)
- [HTMX SSE extension](https://htmx.org/extensions/server-sent-events/)
- [Alpine.js docs](https://alpinejs.dev/start-here)
- [HTMX + Rails tutorial (Evil Martians)](https://evilmartians.com/chronicles/htmx-in-rails)
- [The Hypermedia Systems book](https://hypermedia.systems) — free online; written by the HTMX author
