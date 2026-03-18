# Pagination

Loading message history efficiently without fetching everything at once.

---

## The Problem

A room with years of history might have hundreds of thousands of messages. Loading all of them
on page load is not viable. Chat apps need:

1. **Initial load** — show the most recent N messages
2. **Load more** — scroll up to fetch older messages ("infinite scroll upward")
3. **Jump to date / search result** — load messages around a specific point in time

---

## Approaches

### Option 1: Cursor-based Pagination (Recommended)

Use the message `id` or `created_at` as a cursor. Fetch messages before or after a given point.

```ruby
# Load most recent 50 messages
@messages = @room.messages.order(created_at: :desc).limit(50).reverse

# Load 50 messages before a given message (user scrolled up)
@messages = @room.messages
  .where("created_at < ?", params[:before])
  .order(created_at: :desc)
  .limit(50)
  .reverse
```

**Why cursor over offset:** `OFFSET N` requires the database to scan and discard N rows.
For large offsets (page 500 of a busy channel), this is slow. Cursor-based pagination uses
an indexed column and is O(1) regardless of position.

### Option 2: Offset Pagination (Avoid for large datasets)

```ruby
# Pagy gem
@pagy, @messages = pagy(@room.messages.order(:created_at), items: 50)
```

Works fine for small message counts. Degrades at scale. Not suitable for infinite scroll.

---

## Loading Earlier Messages (Scroll Up)

Use a Turbo Frame to load earlier messages without a full page reload:

```erb
<%# Top of message list — link to load earlier messages %>
<% if @has_earlier_messages %>
  <%= turbo_frame_tag "earlier-messages", src: room_messages_path(@room, before: @messages.first.created_at) do %>
    <div id="load-earlier">
      <%= link_to "Load earlier messages", "#", data: { turbo_frame: "earlier-messages" } %>
    </div>
  <% end %>
<% end %>

<div id="messages">
  <%= render @messages %>
</div>
```

Or trigger automatically when the user scrolls to the top (Stimulus + IntersectionObserver):

```javascript
// app/javascript/controllers/infinite_scroll_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { url: String }

  connect() {
    this.observer = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) this.loadEarlier()
    })
    this.observer.observe(this.element)
  }

  loadEarlier() {
    this.observer.disconnect()
    fetch(this.urlValue, { headers: { "Turbo-Frame": "earlier-messages" } })
      .then(r => r.text())
      .then(html => {
        // prepend messages, update scroll position
      })
  }
}
```

---

## Scroll Position on Load

The message list should scroll to the bottom on initial load, and maintain position when
older messages are prepended (so the user doesn't lose their place).

```javascript
// app/javascript/controllers/message_list_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    this.scrollToBottom()
  }

  scrollToBottom() {
    this.element.scrollTop = this.element.scrollHeight
  }

  // Call before prepending older messages to preserve scroll position
  preserveScrollPosition(callback) {
    const previousHeight = this.element.scrollHeight
    callback()
    this.element.scrollTop += this.element.scrollHeight - previousHeight
  }
}
```

---

## Database Indexes

Essential for performant pagination:

```ruby
# Order by created_at in both directions
add_index :messages, [:room_id, :created_at]
add_index :direct_messages, [:conversation_id, :created_at]
```

---

## Initial Page Load Strategy

Load the last 50 messages. If the room has fewer than 50, show all. Keep the query fast:

```ruby
def index
  @messages = @room.messages
    .includes(:user, :reactions)     # avoid N+1
    .order(created_at: :desc)
    .limit(50)
    .reverse

  @has_earlier_messages = @room.messages.count > 50
end
```

---

## References

- [Pagy gem](https://github.com/ddnexus/pagy)
- [Cursor pagination explained](https://use-the-index-luke.com/no-offset)
- [IntersectionObserver MDN](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API)
