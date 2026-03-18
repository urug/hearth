# Action Cable Authentication

Action Cable connections are not automatically authenticated. This is the single most common
security mistake in Rails real-time apps — forgetting that WebSocket connections operate
outside the normal Rails request/response cycle and must be authenticated separately.

---

## The Problem

A regular Rails controller has `before_action :authenticate_user!` protecting it. Action Cable
channels do not. If you forget to authenticate the connection, **any visitor can subscribe to
any channel** — including private room streams and DM conversations.

```ruby
# WRONG — unauthenticated connection
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    # Nothing here — anyone can connect
  end
end
```

---

## Authenticating the Connection

Authentication happens in `ApplicationCable::Connection`, which runs once when the WebSocket
connection is established. Reject unauthenticated connections here — before any channel code runs.

### With Rails 8 Auth Generator

The Rails 8 auth generator creates a `Session` model. Find the session from a signed cookie:

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private

    def find_verified_user
      if session = Session.find_by(id: cookies.signed[:session_id])
        session.user
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

### With Devise

```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private

    def find_verified_user
      if verified_user = env["warden"].user
        verified_user
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

`env["warden"]` gives you access to Devise's Warden middleware, which has already run by
the time the WebSocket handshake happens.

---

## Authorizing Channel Subscriptions

Authenticating the connection establishes *who* is connected. You still need to authorize
*what* they can subscribe to in each channel.

```ruby
# app/channels/room_channel.rb
class RoomChannel < ApplicationCable::Channel
  def subscribed
    room = Room.find(params[:room_id])

    # Reject if user is not a member of this room
    reject unless current_user.member_of?(room)

    stream_for room
  end
end
```

`reject` closes the subscription cleanly. Without it, any authenticated user could subscribe
to any room stream regardless of membership.

---

## The Two Levels

| Level | Where | What it prevents |
|---|---|---|
| **Connection auth** | `ApplicationCable::Connection` | Unauthenticated visitors connecting at all |
| **Channel auth** | Each `Channel#subscribed` | Authenticated users subscribing to streams they shouldn't see |

Both are required. Connection auth alone is not enough.

---

## identified_by

`identified_by :current_user` does two things:

1. Makes `current_user` available in all channel instances
2. Stores the identifier so Action Cable can look up connections by user — used for
   broadcasting to a specific user:

```ruby
# Broadcast to a specific user from anywhere in the app
ActionCable.server.broadcast(
  "user_#{user.id}",
  { type: "notification", message: "You were mentioned" }
)
```

---

## Testing Connection Auth

```ruby
# test/channels/application_cable/connection_test.rb
class ConnectionTest < ActionCable::Connection::TestCase
  test "connects with valid session" do
    connect cookies: { session_id: sessions(:alice_session).signed_id }
    assert_equal users(:alice), connection.current_user
  end

  test "rejects connection without session" do
    assert_reject_connection { connect }
  end
end
```

---

## Common Mistakes

| Mistake | Consequence |
|---|---|
| Empty `Connection#connect` | Any visitor can subscribe to any stream |
| Auth in `subscribed` but not `Connection` | Still open to unauthenticated connections; just fails silently on subscribe |
| Trusting `params[:user_id]` from the client | Client can forge any user ID |
| No `reject` in channel subscriptions | Members can subscribe to rooms they're not in |

---

## References

- [Action Cable — Connection docs](https://guides.rubyonrails.org/action_cable_overview.html#server-side-components-connections)
- [ActionCable::Connection::TestCase](https://api.rubyonrails.org/classes/ActionCable/Connection/TestCase.html)
