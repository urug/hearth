# Architecture: Pure SSE (No WebSockets)

## Overview

Server-Sent Events (SSE) is a browser-native protocol for one-directional server-to-client
streaming over a plain HTTP connection. The client opens a persistent `text/event-stream`
connection; the server pushes named events down it whenever something happens.

For a chat app, this is surprisingly viable: messages flow server → client via SSE, and client
→ server via normal HTTP POST. No WebSockets, no Action Cable, no Solid Cable — just HTTP.

## How It Works

```
Client opens SSE connection:
  GET /channels/42/stream
  Accept: text/event-stream
  ← 200, Content-Type: text/event-stream (connection stays open)

User sends a message:
  POST /channels/42/messages   (normal form submit or fetch)
  ← 201

Server broadcasts to all SSE subscribers:
  event: new_message
  data: <div class="message">...</div>

  (each open /stream connection receives this)

Browser receives event and updates DOM.
```

In Rails, `ActionController::Live` enables SSE responses:

```ruby
class ChannelStreamController < ApplicationController
  include ActionController::Live

  def show
    response.headers['Content-Type'] = 'text/event-stream'
    response.headers['Last-Modified'] = Time.now.httpdate

    sse = SSE.new(response.stream, retry: 300, event: "heartbeat")

    MessageBroadcaster.subscribe(params[:id]) do |message_html|
      sse.write(message_html, event: "new_message")
    end
  rescue ActionController::Live::ClientDisconnected
    # client navigated away
  ensure
    sse.close
  end
end
```

For HTMX clients, the SSE extension handles subscription and DOM swapping automatically (see
`alpine-htmx.md`). For Hotwire clients, Turbo can consume SSE via `<turbo-stream-source>`.

## Stack

| Layer | Technology |
|-------|-----------|
| Server push | `ActionController::Live` + Ruby `SSE` class (built into Rails) |
| Client subscription | Native `EventSource` JS API, HTMX SSE extension, or Turbo Stream Source |
| Message broadcasting | In-process pub/sub (e.g. `Concurrent::Channel`) or Postgres `LISTEN/NOTIFY` |
| No Action Cable | Not required |
| No Redis | Not required |

## Pros

- **Minimal dependencies** — No Action Cable, no Solid Cable, no Redis, no WebSocket protocol.
  Just HTTP.
- **Debuggable** — SSE streams appear in browser DevTools as regular HTTP responses. Easy to
  inspect events.
- **Stateless-ish** — Each SSE connection is independent. No session state on the wire.
- **Works through proxies and load balancers** — HTTP/1.1 streaming is well-understood by
  infrastructure. WebSockets sometimes require special proxy config.
- **Great for discussion** — Illustrates the minimum viable real-time pattern. Good pedagogical
  value for a group learning real-time Rails.
- **Browser native** — `EventSource` is built into every browser. No JS library needed on the
  client.

## Cons

- **One thread per connection** — `ActionController::Live` blocks a Puma thread for each open
  SSE connection. With Puma's default of 5 threads, you can serve 5 concurrent streams before
  queueing. Mitigations: increase threads, use Falcon (async), or use Postgres `LISTEN/NOTIFY`
  with a dedicated thread pool.
- **Broadcasting requires coordination** — Unlike Action Cable (which has built-in pub/sub),
  you must implement your own mechanism to fan out a message to all open SSE connections for a
  channel. Common approaches:
  - In-process registry (works on single server, breaks with multiple dynos)
  - Postgres `LISTEN/NOTIFY` (works across multiple servers, elegant)
- **No reconnect state** — If the connection drops, `EventSource` reconnects automatically but
  may miss messages. Requires a `Last-Event-ID` implementation to replay missed events.
- **One-directional only** — SSE cannot send data from server to client bidirectionally.
  For chat this is fine (POST handles client → server), but presence/typing indicators require
  careful design.

## Postgres LISTEN/NOTIFY Pattern

A clean solution for fan-out that works across multiple servers:

```ruby
# When a message is saved:
ActiveRecord::Base.connection.execute(
  "NOTIFY channel_42, '#{message_html.to_json}'"
)

# In the SSE controller, each connection listens:
conn = ActiveRecord::Base.connection_pool.checkout
conn.execute("LISTEN channel_42")
loop do
  conn.raw_connection.wait_for_notify(5) do |_, _, payload|
    sse.write(payload, event: "new_message")
  end
end
```

This keeps fan-out in Postgres — no Redis, no in-process registry, works across dynos.

## Fit for Hearth

Strong pedagogical fit. This is the simplest possible real-time architecture and makes the
mechanics visible. For a group that wants to understand how real-time works before reaching for
a framework, SSE-only is an excellent starting point.

Production viability depends on expected concurrency. For a small group (URUG has ~100 members),
the thread scaling concern is manageable. For larger audiences, the Postgres `LISTEN/NOTIFY`
pattern combined with a larger thread pool handles it cleanly.

## References

- [ActionController::Live docs](https://api.rubyonrails.org/classes/ActionController/Live.html)
- [MDN: Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [Postgres LISTEN/NOTIFY](https://www.postgresql.org/docs/current/sql-listen.html)
- [Falcon web server](https://github.com/socketry/falcon) — async Ruby server, eliminates the
  thread-per-connection problem
- [Turbo Stream Source](https://turbo.hotwired.dev/reference/streams#streaming-from-http) —
  consume SSE streams natively in Turbo without HTMX
