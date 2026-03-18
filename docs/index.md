# Hearth Docs

Research and reference material for building Hearth — a self-hosted group chat for URUG.

---

## Start Here

| Doc | What it covers |
|---|---|
| [Scope](./scope.md) | What v1 is, what it isn't, definition of done |
| [Features](./features.md) | Full feature list organized by priority |
| [Models](./models.md) | Data model, relationships, ER diagrams |

---

## Architecture

Comparison of approaches for the real-time stack.

| Doc | Summary |
|---|---|
| [Architecture Index](./architectures/index.md) | Tradeoffs table for all options |
| [Hotwire](./architectures/hotwire.md) | Turbo Streams + Action Cable — the Rails default |
| [Alpine.js + HTMX](./architectures/alpine-htmx.md) | Hypermedia-driven, framework-agnostic |
| [Inertia.js](./architectures/inertia.md) | Rails routing + component-based frontend |
| [Rails API + SPA](./architectures/rails-api-spa.md) | Fully decoupled JSON API + JS frontend |
| [Pure SSE](./architectures/sse-only.md) | Simplest possible real-time — no Action Cable |
| [Islands](./architectures/islands.md) | Hotwire + targeted JS components for complex UI |
| [Phlex](./architectures/phlex.md) | Ruby-based view layer, replaces ERB |

---

## Implementation Topics

| Doc | What it covers |
|---|---|
| [Authentication](./authentication.md) | Rails 8 auth generator vs Devise, GitHub OAuth |
| [Authorization](./authorization.md) | Pundit, Action Policy, room membership enforcement |
| [Action Cable Auth](./action-cable-auth.md) | Authenticating WebSocket connections — easy to get wrong |
| [Message Formatting](./message-formatting.md) | Markdown, syntax highlighting, link previews |
| [File Uploads](./file-uploads.md) | Active Storage, direct uploads, S3 |
| [Search](./search.md) | pg_search vs Meilisearch |
| [Pagination](./pagination.md) | Cursor-based pagination, infinite scroll upward |
| [Presence](./presence.md) | Online indicators, typing indicators |
| [Notifications](./notifications.md) | In-app, email, batching |
| [Background Jobs](./background-jobs.md) | Solid Queue, job patterns, recurring jobs |
| [Performance](./performance.md) | N+1 queries, caching, indexes |

---

## Operations

| Doc | What it covers |
|---|---|
| [Deployment](./deployment.md) | Kamal, server requirements, CI/CD |
| [AnyCable](./anycable.md) | Scaling Action Cable beyond a single server |
| [Testing](./testing.md) | Test strategy, Action Cable testing, CI setup |
| [Contributing](./contributing.md) | Local setup, running tests, code style |

---

## Reference

| Doc | What it covers |
|---|---|
| [References](./references.md) | Code to study, key gems, tutorials, open source comparisons |
