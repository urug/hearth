# AnyCable

A drop-in replacement for Action Cable that dramatically improves WebSocket performance.

---

## What It Is

Action Cable handles WebSocket connections in Ruby threads inside the Rails process. Under load,
this consumes significant memory — each connection holds a Ruby thread and associated state.

AnyCable offloads the WebSocket connection handling to a separate Go process (`anycable-go`).
The Go process holds connections cheaply (goroutines use ~8KB vs ~1MB for Ruby threads), and
proxies incoming messages to Rails via gRPC.

```
Browser <──WebSocket──> anycable-go (Go, lightweight)
                              │
                           gRPC
                              │
                        Rails (only handles RPC calls, not persistent connections)
```

---

## Performance Comparison

| | Action Cable | AnyCable |
|---|---|---|
| Memory per connection | ~1–2 MB | ~8 KB |
| Connections on 512MB RAM | ~250 | ~50,000+ |
| Throughput | Limited by Ruby threads | Near-native Go |
| Infrastructure | Single Rails process | Rails + anycable-go |

For URUG scale (~100 users), Action Cable is fine. AnyCable becomes relevant when you have
thousands of concurrent connections.

---

## Setup

```ruby
gem "anycable-rails"
```

```yaml
# config/anycable.yml
development:
  broadcast_adapter: http

production:
  broadcast_adapter: http
  http_broadcast_url: http://localhost:8090/_broadcast
```

Run the Go server alongside Rails:

```bash
anycable-go --port=8080
```

The Rails app still defines channels normally — AnyCable is transparent to channel code.

---

## Solid Cable vs AnyCable

| | Solid Cable | AnyCable |
|---|---|---|
| Backend | Postgres | Go process |
| Infrastructure | None (uses existing DB) | Extra process |
| Memory efficiency | Rails-level | Very high |
| Setup complexity | Zero | Moderate |
| Best for | Small–medium scale | Large scale |

Hearth ships with Solid Cable. AnyCable is an upgrade path, not a starting point.

---

## When to Switch

Consider AnyCable when:

- Memory usage from Action Cable connections is measurable
- Concurrent user count regularly exceeds a few hundred
- You need horizontal scaling across multiple servers (AnyCable Go handles fan-out)
- You want built-in presence support (AnyCable Pro)

---

## Kamal Deployment with AnyCable

Add `anycable-go` as a second service in Kamal:

```yaml
# config/deploy.yml
accessories:
  anycable:
    image: anycable/anycable-go:latest
    host: your-server-ip
    port: 8080
    env:
      clear:
        ANYCABLE_BROADCAST_ADAPTER: http
        ANYCABLE_HTTP_BROADCAST_URL: http://localhost:8090/_broadcast
        ANYCABLE_RPC_HOST: localhost:50051
```

---

## References

- [AnyCable docs](https://docs.anycable.io)
- [anycable-rails gem](https://github.com/anycable/anycable-rails)
- [anycable-go](https://github.com/anycable/anycable-go)
- [AnyCable vs Action Cable benchmark](https://anycable.io/#benchmarks)
