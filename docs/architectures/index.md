# Architecture Options

Comparison of approaches for building Hearth. Each option is documented in its own file.

## Tradeoffs at a Glance

| Architecture | Real-time | JS Complexity | Build Step | API Layer | Mobile-Ready | Best For |
|---|---|---|---|---|---|---|
| [Hotwire](./hotwire.md) | Action Cable + Turbo Streams | Low | No | No | No | Ruby-first teams, fast iteration |
| [Rails API + SPA](./rails-api-spa.md) | WebSockets or SSE via API | High | Yes | Yes | Yes | Teams that want a native app later |
| [Inertia.js](./inertia.md) | Action Cable + Turbo/Pusher | Medium | Yes | No | No | Teams that want modern JS with Rails routing |
| [Alpine.js + HTMX](./alpine-htmx.md) | SSE or Action Cable | Low | No | No | No | Hypermedia purists, framework-agnostic approach |
| [Pure SSE](./sse-only.md) | ActionController::Live + EventSource | Minimal | No | No | No | Simplest possible real-time; great for learning |
| [Phlex](./phlex.md) | Pairs with any above | Low | No | No | No | Ruby-centric view layer; replaces ERB with Ruby objects |
| [Islands](./islands.md) | Hotwire/HTMX + targeted JS components | Low–Medium | Yes (for islands only) | No | Partial | Hotwire-first with escape hatches for complex UI |

## Key Considerations

| Consideration | Notes |
|---|---|
| **Audience** | URUG is a Ruby group — less JS overhead is a feature, not a limitation |
| **Infrastructure** | Solid Cable eliminates Redis; Postgres handles Action Cable state |
| **History retention** | All options support unlimited history — this is a data model concern, not an architecture one |
| **Deployment** | Kamal is already configured; all options deploy the same way |
| **Mobile app** | Not in scope for v1, but worth noting which options make it easier later |
| **Onboarding** | The easier it is for contributors to understand the stack, the more people can help |

## Recommendation

Hotwire is the natural fit: the Gemfile already includes Solid Cable, the group knows Ruby,
and it is the stack Basecamp uses for their own Campfire product. The other options are worth
understanding, but they introduce complexity (Node builds, API design, CORS, client-side state
management) that is not justified by what Hearth needs to do.
