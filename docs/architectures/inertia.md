# Architecture: Inertia.js

## Overview

Inertia.js is a protocol that lets you build single-page-app style interfaces using server-side
routing and controllers. You write Rails controllers that return Inertia responses (instead of
HTML or JSON), and the frontend renders them using a component framework — React, Vue, or Svelte.

The result feels like a modern SPA (no full page reloads, fast navigation, component-based UI)
without building a separate API. Rails still owns routing and authorization. The frontend just
renders what the server tells it to.

For real-time, Inertia has no built-in solution — you pair it with Action Cable, Pusher, or SSE
for live updates.

## How It Works

1. First request: Rails renders a full HTML page with a root `<div data-page="...">` containing
   the serialized page props as JSON.
2. Subsequent navigations: Inertia intercepts link clicks and form submissions, makes an XHR
   request, and Rails returns a JSON response with the next page's component name and props.
3. Inertia swaps the component on the client without a full page reload.

```
First load:
  Browser → GET /channels/42
          ← HTML (with <div data-page='{"component":"Channel","props":{...}}'>)

Navigation:
  Inertia → GET /channels/43  (X-Inertia: true header)
           ← JSON {"component":"Channel","props":{...}}
  Inertia swaps component, no page reload
```

For real-time messages, a separate Action Cable subscription pushes new message data to the
component, which updates its local state.

## Stack

| Layer | Technology |
|-------|-----------|
| Server | Rails (controllers return Inertia responses via `inertia-rails` gem) |
| Protocol | Inertia.js |
| Frontend framework | React, Vue 3, or Svelte (team's choice) |
| Real-time transport | Action Cable (or Pusher/Ably as a managed alternative) |
| Asset bundling | Vite via `vite_ruby` gem (recommended) or jsbundling-rails |
| CSS | Tailwind, CSS Modules, or plain CSS |

Relevant gems: [`inertia-rails`](https://github.com/inertiajs/inertia-rails)

## Pros

- **No API design required** — Rails controllers return data directly as props; no JSON API
  spec, no versioning, no CORS configuration.
- **Component-based UI** — React/Vue/Svelte components are more expressive for complex
  interactions (emoji pickers, drag-and-drop, rich text editors) than Stimulus.
- **Familiar to JS developers** — Contributors who know React or Vue can contribute without
  learning Hotwire.
- **Strong TypeScript support** — Props from Rails can be typed end-to-end with tools like
  `inertia-rails` + TypeScript interfaces.
- **File-based routing optional** — Can use Rails routes (conventional) or add a JS router on
  top.

## Cons

- **Node build step required** — Vite (or webpack) must run alongside Rails. More moving parts
  in development and CI.
- **Real-time is DIY** — Inertia has no streaming primitive. You must wire up Action Cable (or
  Pusher) manually and manage component state for live updates.
- **Two rendering worlds** — Server-side state (Rails) and client-side state (React/Vue) can
  diverge; requires discipline to keep in sync.
- **Less idiomatic for Ruby shops** — The frontend logic is in JS/TS, which may be unfamiliar
  to Ruby-first contributors.
- **SSR is complex** — Server-side rendering for SEO/performance requires running a Node
  process alongside Rails.

## Fit for Hearth

Moderate fit. Inertia removes the pain of building a full API while still enabling component-
based UI. The tradeoff is a build step and a split mental model between server and client state.
For a group chat app, the real-time wiring is non-trivial and requires custom Action Cable
channel management on both sides.

Worth considering if the group has Vue or React experience and wants to use it, or if richer
client-side interactions (beyond what Stimulus can cleanly handle) are a priority.

## Real-Time Pattern with Action Cable

```javascript
// In a Vue/React component
import { usePage } from '@inertiajs/vue3'
import { onMounted, onUnmounted, ref } from 'vue'

const messages = ref(page.props.messages)
let subscription

onMounted(() => {
  subscription = App.cable.subscriptions.create(
    { channel: 'RoomChannel', room_id: props.channel.id },
    { received(data) { messages.value.push(data) } }
  )
})

onUnmounted(() => subscription.unsubscribe())
```

## References

- [Inertia.js docs](https://inertiajs.com)
- [inertia-rails gem](https://github.com/inertiajs/inertia-rails)
- [vite_ruby](https://vite-ruby.netlify.app) — recommended Vite integration for Rails
- [PingCRM demo](https://github.com/ledermann/pingcrm-vue) — Rails + Inertia + Vue reference app
