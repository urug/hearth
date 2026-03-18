# Architecture: Rails API + SPA

## Overview

Rails serves a JSON API only — no HTML rendering except for the initial shell page. A separate
JavaScript frontend (React, Vue, Svelte, etc.) handles all UI. The two communicate over HTTP
(REST or GraphQL) and WebSockets (Action Cable or a third-party service like Pusher).

This is the most decoupled option. Frontend and backend can be developed, deployed, and scaled
independently. It is also the most complex to set up and maintain.

## How It Works

```
Initial load:
  Browser → GET /
           ← HTML shell (one <div id="app">, JS bundle)
  JS app boots, fetches initial data

API calls:
  SPA → GET /api/v1/channels/42/messages
      ← JSON { messages: [...] }
  SPA renders messages from JSON

Real-time:
  Option A — Action Cable WebSocket:
    SPA connects to ws://app/cable
    Subscribes to RoomChannel
    Server broadcasts JSON payloads on new messages

  Option B — Pusher/Ably (managed WebSockets):
    Rails triggers Pusher event on message create
    SPA receives event via Pusher JS client
    No Action Cable needed
```

## Stack

| Layer | Technology |
|-------|-----------|
| Backend | Rails in API mode (`rails new --api`) |
| Serialization | `jsonapi-serializer`, `alba`, or `jbuilder` |
| Authentication | JWT (stateless) or session cookies with CSRF handling |
| Real-time | Action Cable (self-hosted) or Pusher/Ably (managed) |
| Frontend framework | React, Vue 3, Svelte, or SolidJS |
| Frontend routing | React Router, Vue Router, TanStack Router |
| State management | Zustand, Pinia, TanStack Query, or Redux |
| Build tool | Vite (standalone, separate repo or monorepo) |
| API contract | REST or GraphQL (`graphql-ruby`) |

## Pros

- **Mobile-ready** — The API works equally well for a web SPA, iOS app, or Android app. Best
  option if native mobile is a future goal.
- **Independent deployability** — Frontend can be deployed to a CDN (Vercel, Netlify, S3+CF);
  backend scales separately.
- **Frontend flexibility** — Use any JS framework, tooling, or component library without
  constraints from Rails conventions.
- **Clear contract** — API versioning and documentation (OpenAPI/Swagger) make the boundary
  explicit. Good for teams with separate frontend/backend contributors.
- **Rich client-side interactions** — Complex UI (virtual scrolling through message history,
  drag-and-drop, offline support) is more natural in a full JS runtime.

## Cons

- **Most complex setup** — CORS, authentication across origins, API versioning, separate build
  pipelines, and two codebases to maintain.
- **No Rails conventions for the frontend** — You leave the Rails "pit of success" and own all
  frontend architecture decisions.
- **Client-side state management** — Messages, channels, presence, and notifications must all
  be managed in JS. This is significant complexity for a chat app.
- **Auth is harder** — Session cookies don't work cleanly across origins. JWT adds complexity
  (refresh tokens, storage, revocation). `rack-cors` required.
- **Overkill for Hearth v1** — The benefits (mobile app, CDN deployment) are not in scope.
  The costs are immediate.

## Authentication Options

| Approach | Pros | Cons |
|---|---|---|
| JWT | Stateless, works cross-origin | Revocation is hard; refresh token complexity |
| Cookie sessions + CORS | Familiar Rails pattern | Requires careful CORS/SameSite config |
| Devise + `devise-jwt` | Batteries included | More gems, more config |

## Real-Time Options

| Approach | Pros | Cons |
|---|---|---|
| Action Cable | Self-hosted, no cost, already in Rails | Stateful server, harder to scale horizontally |
| Pusher | Managed, generous free tier, great DX | External dependency, cost at scale |
| Ably | Managed, stronger guarantees than Pusher | Cost, another vendor |
| Supabase Realtime | Postgres-native, open source option | Adds a separate infrastructure component |

## Fit for Hearth

Poor fit for v1. The decoupled architecture solves problems Hearth doesn't have yet (mobile app,
separate teams, CDN-deployed frontend) while adding real costs today (CORS, auth complexity,
client-side state management, two build pipelines).

The one scenario where this makes sense: if the group explicitly wants to learn or demonstrate
modern SPA architecture as part of the project goals, rather than ship a working chat app as
quickly as possible.

## References

- [Rails API mode guide](https://guides.rubyonrails.org/api_app.html)
- [rack-cors gem](https://github.com/cyu/rack-cors)
- [alba serializer](https://github.com/okuramasafumi/alba) — fast, lightweight JSON serialization
- [TanStack Query](https://tanstack.com/query) — server state management for React/Vue/Svelte
- [Pusher](https://pusher.com) — managed WebSocket service with a free tier
