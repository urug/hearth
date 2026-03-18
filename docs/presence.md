# Presence & Typing Indicators

Showing who is online and who is currently typing.

---

## Online Presence

### Approach 1: Heartbeat via Action Cable (Recommended)

When a user connects to Action Cable, mark them online. When they disconnect or a heartbeat
times out, mark them offline.

```ruby
# app/channels/presence_channel.rb
class PresenceChannel < ApplicationCable::Channel
  def subscribed
    Current.user.update!(last_seen_at: Time.current, online: true)
    stream_from "presence"
    ActionCable.server.broadcast("presence", {
      type: "online",
      user_id: Current.user.id
    })
  end

  def unsubscribed
    Current.user.update!(online: false)
    ActionCable.server.broadcast("presence", {
      type: "offline",
      user_id: Current.user.id
    })
  end

  def heartbeat
    Current.user.update!(last_seen_at: Time.current)
  end
end
```

Client sends a heartbeat every 30 seconds:
```javascript
// app/javascript/channels/presence_channel.js
const channel = consumer.subscriptions.create("PresenceChannel", {
  connected() {
    this.heartbeatTimer = setInterval(() => this.perform("heartbeat"), 30000)
  },
  disconnected() {
    clearInterval(this.heartbeatTimer)
  }
})
```

### Approach 2: last_seen_at Polling

Simpler: just update `last_seen_at` on every request via a `before_action`. Define "online" as
active within the last N minutes.

```ruby
# app/controllers/application_controller.rb
before_action :touch_last_seen

def touch_last_seen
  Current.user&.update_column(:last_seen_at, Time.current)
end
```

```ruby
# app/models/user.rb
def online?
  last_seen_at.present? && last_seen_at > 5.minutes.ago
end
```

No real-time push — presence indicators update on next page load or Turbo navigation.

### Approach 3: Postgres LISTEN/NOTIFY

Broadcast presence changes via Postgres notify, fan out via SSE or Action Cable. Works across
multiple servers without a shared in-memory store.

---

## Recommendation

**Approach 2 (last_seen_at polling)** for v1 — zero infrastructure, trivially simple. Add
Action Cable heartbeats when real-time presence indicators are a priority.

---

## Typing Indicators

Show "Alice is typing..." when a user is composing a message.

### Implementation

```ruby
# app/channels/typing_channel.rb
class TypingChannel < ApplicationCable::Channel
  def subscribed
    stream_from "typing:room:#{params[:room_id]}"
  end

  def typing
    ActionCable.server.broadcast(
      "typing:room:#{params[:room_id]}",
      {
        user_id: Current.user.id,
        name: Current.user.display_name,
        typing: true
      }
    )
  end

  def stopped_typing
    ActionCable.server.broadcast(
      "typing:room:#{params[:room_id]}",
      { user_id: Current.user.id, typing: false }
    )
  end
end
```

Client fires `typing` on keydown, `stopped_typing` on submit or after 3 seconds of inactivity:
```javascript
let typingTimer
messageInput.addEventListener("keydown", () => {
  channel.perform("typing")
  clearTimeout(typingTimer)
  typingTimer = setTimeout(() => channel.perform("stopped_typing"), 3000)
})
```

### Considerations

- Debounce aggressively — don't broadcast on every keystroke
- Never show your own typing indicator back to yourself
- Auto-expire after 5 seconds server-side to handle dropped connections
- Multiple typers: "Alice and Bob are typing..." (collect active typers in a Set)

---

## Presence at Scale

For a single-server deployment (URUG scale), in-memory state or `last_seen_at` is fine.
For multi-server deployments, presence state must be shared:

| Approach | How |
|---|---|
| Database column | `users.online` boolean + `last_seen_at` — works everywhere |
| Redis | Shared in-memory store; fast but adds infrastructure |
| Postgres LISTEN/NOTIFY | Pub/sub without Redis; works across servers |
| AnyCable | Built-in presence support in AnyCable Pro |

---

## References

- [Action Cable docs](https://guides.rubyonrails.org/action_cable_overview.html)
- [Turbo + presence patterns (37signals blog)](https://dev.37signals.com)
