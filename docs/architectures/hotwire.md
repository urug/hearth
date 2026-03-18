# Architecture: Full Hotwire Stack

## Overview

The "Rails way" for Hearth. Use Turbo Streams over Action Cable to push real-time HTML fragments
to connected clients. Stimulus handles any client-side interactivity (scroll behavior, emoji
pickers, etc.). No separate frontend build step, no API layer — just Rails rendering HTML.

Rails 8 ships with this stack by default. Hearth already has `solid_cable` in the Gemfile, which
replaces Redis with a Postgres-backed Action Cable adapter.

## How Real-Time Works

1. User posts a message → form submits to `MessagesController#create`
2. Controller saves the message and broadcasts a Turbo Stream to the channel's stream
3. All connected clients receive the HTML fragment and Turbo appends it to the message list
4. No polling, no websocket boilerplate — Action Cable + Turbo handle the plumbing

```
Browser → POST /channels/:id/messages
       ← 200 (empty or redirect)

Action Cable broadcasts to "channel_42":
  <turbo-stream action="append" target="messages">
    <template>...rendered message partial...</template>
  </turbo-stream>

All subscribed browsers receive and apply the update automatically.
```

## Stack

| Layer | Technology |
|-------|-----------|
| Real-time transport | Action Cable (Solid Cable adapter — Postgres, no Redis) |
| HTML streaming | Turbo Streams |
| Navigation | Turbo Drive + Turbo Frames |
| Client interactivity | Stimulus |
| Asset pipeline | Propshaft (already in Gemfile) |
| JS bundling | Import maps (no Node/webpack needed) or jsbundling-rails if needed |

## Pros

- **Minimal JS** — Most logic stays in Ruby. Easier to onboard Rails developers.
- **No build step** — Import maps ship with Rails 8; no Node, webpack, or Vite required.
- **Already wired** — `solid_cable` is in the Gemfile; Action Cable is ready to go.
- **Server controls HTML** — Authorization, formatting, and rendering all stay in one place.
- **Solid Cable = no Redis** — One less infrastructure dependency; cable state lives in Postgres.
- **Battle-tested for chat** — Basecamp/Hey are built this way; HEY World, Campfire (the real one) use this pattern.

## Cons

- **Complex interactions need more Stimulus** — Things like emoji autocomplete, drag-and-drop,
  or a floating toolbar require more client-side code than a SPA would.
- **Optimistic UI is harder** — Messages don't appear instantly before the server confirms;
  requires extra effort to fake local rendering.
- **Mobile app story is weak** — If a native iOS/Android app is ever needed, this architecture
  doesn't produce an API naturally.
- **Turbo Streams debugging** — Tracing why a stream update didn't fire can be opaque compared
  to a JSON API with clear request/response cycles.

## Fit for Hearth

Strong fit. URUG is a Ruby group — keeping logic in Ruby plays to the audience. The existing
Gemfile already has everything needed. A real-time chat app is exactly the use case Turbo Streams
were designed for, and Basecamp's own Campfire product uses this stack.

## Key Implementation Patterns

### Broadcasting messages
```ruby
# app/models/message.rb
after_create_commit -> {
  broadcast_append_to channel, target: "messages", partial: "messages/message"
}
```

### Subscribing in the view
```erb
<%= turbo_stream_from @channel %>
<div id="messages">
  <%= render @channel.messages %>
</div>
```

### Scroll-to-bottom on new message (Stimulus)
A small Stimulus controller observes DOM mutations on `#messages` and scrolls to the bottom
when new content is appended.

## References

- [Turbo Handbook](https://turbo.hotwired.dev/handbook/introduction)
- [Solid Cable README](https://github.com/rails/solid_cable)
- [Basecamp Campfire](https://once.com/campfire) — commercial product built on this exact stack
- [HEY source patterns](https://dev.37signals.com) — 37signals engineering blog
