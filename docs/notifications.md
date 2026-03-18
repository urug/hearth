# Notifications

Alerting users to activity they care about.

---

## Notification Types

| Trigger | Channel |
|---|---|
| `@username` mention in a room | In-app + email |
| DM received | In-app + email |
| Reply in a thread you participated in | In-app + email |
| Room invitation | Email |
| Missed activity digest | Email (daily/weekly) |

---

## In-App Notifications

### Model

```ruby
# app/models/notification.rb
class Notification < ApplicationRecord
  belongs_to :user
  belongs_to :notifiable, polymorphic: true  # Message, Conversation, etc.

  scope :unread, -> { where(read_at: nil) }
end
```

Migration:
```ruby
create_table :notifications do |t|
  t.references :user, null: false, foreign_key: true
  t.references :notifiable, polymorphic: true, null: false
  t.string :action, null: false   # "mention", "dm", "thread_reply"
  t.datetime :read_at
  t.timestamps
end
```

### Real-Time Delivery

Broadcast to the user's personal notification stream via Action Cable:

```ruby
# app/models/notification.rb
after_create_commit -> {
  broadcast_prepend_to(
    "notifications:#{user_id}",
    target: "notifications",
    partial: "notifications/notification"
  )
}
```

```erb
<%# In the layout — subscribes to the current user's notification stream %>
<%= turbo_stream_from "notifications:#{Current.user.id}" %>
<div id="notifications">
  <%= render Current.user.notifications.unread %>
</div>
```

### Unread Count

Display in the sidebar:
```ruby
Current.user.notifications.unread.count
```

Cache this count with Solid Cache to avoid a query on every page load:
```ruby
Rails.cache.fetch("notifications:#{user_id}:unread_count", expires_in: 5.minutes) do
  notifications.unread.count
end
```

---

## Email Notifications

Use Action Mailer + Active Job. Deliver asynchronously — never block a request on email.

```ruby
# app/mailers/notification_mailer.rb
class NotificationMailer < ApplicationMailer
  def mention(user, message)
    @user = user
    @message = message
    @room = message.room

    mail(
      to: @user.email,
      subject: "#{@message.user.name} mentioned you in ##{@room.name}"
    )
  end
end
```

```ruby
# Enqueue from a job triggered after message creation
NotificationMailer.mention(user, message).deliver_later
```

### Batching / Digest

Avoid email spam by batching notifications:

```ruby
# Wait 5 minutes before sending — if the user reads the message in that time, skip
NotificationMailer.mention(user, message).deliver_later(wait: 5.minutes)
```

Or collect them into a digest via a Solid Queue recurring job:

```yaml
# config/recurring.yml
daily_digest:
  class: DailyDigestJob
  schedule: every day at 8am
```

```ruby
# app/jobs/daily_digest_job.rb
class DailyDigestJob < ApplicationJob
  def perform
    User.where(digest_frequency: "daily").find_each do |user|
      next unless user.notifications.unread.any?
      NotificationMailer.digest(user).deliver_later
    end
  end
end
```

---

## Notification Preferences

Users should be able to opt out of specific notification types:

```ruby
# users table additions
t.boolean :notify_mentions, default: true
t.boolean :notify_dms, default: true
t.boolean :notify_threads, default: true
t.string  :digest_frequency, default: "never"  # "daily", "weekly", "never"
```

Check preferences before delivering:
```ruby
def notify_mention(user, message)
  return unless user.notify_mentions?
  NotificationMailer.mention(user, message).deliver_later(wait: 5.minutes)
end
```

---

## Browser Push Notifications

Future consideration. Requires:
- Service worker registration
- Push API (Web Push protocol)
- User permission grant
- `webpush` gem for server-side delivery

Not recommended for v1 — in-app + email covers the basics.

---

## References

- [Action Mailer basics](https://guides.rubyonrails.org/action_mailer_basics.html)
- [Solid Queue recurring jobs](https://github.com/rails/solid_queue#recurring-tasks)
- [webpush gem](https://github.com/zaru/webpush)
