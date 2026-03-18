# Performance & N+1 Queries

Common performance traps in a chat app and how to avoid them.

---

## N+1 Queries in Chat

A chat room renders a list of messages. Each message shows the author's name and avatar.
Without eager loading, that's one query per message:

```ruby
# Bad — 1 query for messages + N queries for users
@messages = @room.messages.limit(50)
# In the view: message.user.name  ← fires a query for each message
```

```ruby
# Good — 2 queries total
@messages = @room.messages.includes(:user).limit(50)
```

### Common N+1 sources in Hearth

| Rendering | Fix |
|---|---|
| `message.user.name` | `includes(:user)` |
| `message.reactions` | `includes(:reactions)` |
| `message.reactions.group_by(&:emoji)` | `includes(reactions: :user)` |
| `room.memberships.count` | `includes(:memberships)` or counter cache |
| `user.avatar` (Active Storage) | `with_attached_avatar` scope |
| `user.rooms` in sidebar | `includes(:rooms)` on current user |

### Active Storage N+1

Active Storage attachments have their own N+1 problem:

```ruby
# Bad
@messages.each { |m| m.attachments.each { ... } }

# Good
@messages = @room.messages.with_attached_attachments.limit(50)
```

---

## Bullet Gem (Development Only)

Automatically detects N+1 queries and suggests fixes:

```ruby
group :development do
  gem "bullet"
end
```

```ruby
# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
end
```

---

## Counter Caches

Avoid `COUNT(*)` queries for frequently-displayed totals:

```ruby
# Unread message count — add counter cache to membership
class Membership < ApplicationRecord
  # instead of: Current.user.rooms.each { |r| r.messages.where("created_at > ?", membership.last_read_at).count }
end
```

For unread counts, a better pattern is to query once and compute in Ruby:

```ruby
def unread_counts_for(user)
  user.memberships
    .joins("LEFT JOIN messages ON messages.room_id = memberships.room_id AND messages.created_at > memberships.last_read_at")
    .group("memberships.room_id")
    .count("messages.id")
end
```

---

## Caching

### Fragment Caching

Cache rendered message partials — messages rarely change after creation:

```erb
<% cache message do %>
  <%= render "messages/message", message: message %>
<% end %>
```

The cache key is based on `message.cache_key_with_version` — automatically invalidates on update.

### Russian Doll Caching

Nest caches for efficiency:

```erb
<%# Room cache expires when any message changes %>
<% cache [@room, @room.messages.maximum(:updated_at)] do %>
  <% @messages.each do |message| %>
    <% cache message do %>
      <%= render message %>
    <% end %>
  <% end %>
<% end %>
```

### Solid Cache

Already in the Gemfile. Postgres-backed, zero infrastructure. Configure in `config/cache.yml`:

```yaml
default: &default
  store_options:
    max_age: <%= 1.week.to_i %>
    max_size: <%= 256.megabytes %>
    namespace: <%= Rails.env %>
```

---

## Database Indexes

Beyond pagination indexes, make sure these exist:

```ruby
# Foreign key lookups (Rails adds these automatically with references columns)
add_index :messages, :user_id
add_index :messages, :room_id
add_index :messages, :parent_id
add_index :reactions, :message_id
add_index :memberships, :user_id
add_index :memberships, :room_id

# Query patterns
add_index :messages, [:room_id, :created_at]   # pagination
add_index :memberships, [:user_id, :room_id], unique: true
add_index :users, :last_seen_at                # presence queries
```

---

## Action Cable Connection Overhead

Each connected browser maintains a WebSocket connection. At URUG scale (~100 users), this is
fine. Things to avoid:

- Don't query the database inside `subscribed` callbacks unnecessarily
- Don't broadcast large payloads — send minimal data, re-render on the client if needed
- Use `broadcast_to` with specific streams rather than global streams

---

## Query Logging in Development

Enable verbose query logging to spot problems early:

```ruby
# config/environments/development.rb
config.log_level = :debug  # shows all SQL queries
```

Or use the `query_count` middleware to add query counts to request logs:

```ruby
gem "rack-mini-profiler"
```

---

## References

- [Bullet gem](https://github.com/flyerhzm/bullet)
- [Rails caching guide](https://guides.rubyonrails.org/caching_with_rails.html)
- [Rack Mini Profiler](https://github.com/MiniProfiler/rack-mini-profiler)
- [Use The Index, Luke](https://use-the-index-luke.com) — SQL indexing guide
