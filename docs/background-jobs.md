# Background Jobs

Work that should not happen in the request/response cycle.

---

## What Needs a Job

| Task | Why async |
|---|---|
| Email notifications | SMTP is slow; user shouldn't wait |
| Link unfurling (preview generation) | HTTP requests to external URLs |
| Push notifications | External API calls |
| Image processing (avatar resize) | CPU-intensive |
| Search index updates | Can lag slightly without user impact |
| Mention extraction and notification routing | Can happen after message saves |

---

## Queue Backend

Hearth already has **Solid Queue** in the Gemfile — the Rails-default queue backend that stores
jobs in Postgres. No Redis, no Sidekiq, no separate process needed (in development).

```ruby
gem "solid_queue" # already present
```

In production, Solid Queue runs as a separate process via the Puma plugin or a dedicated worker:

```ruby
# config/puma.rb — runs queue inline with the web process (development/small production)
plugin :solid_queue
```

Or as a separate process (recommended for production):

```bash
bin/jobs
```

---

## Defining Jobs

```ruby
# app/jobs/notify_mentions_job.rb
class NotifyMentionsJob < ApplicationJob
  queue_as :default

  def perform(message_id)
    message = Message.find(message_id)
    message.mentioned_users.each do |user|
      NotificationMailer.mention(user, message).deliver_later
    end
  end
end
```

Enqueue from a model callback:
```ruby
# app/models/message.rb
after_create_commit -> { NotifyMentionsJob.perform_later(id) }
```

---

## Queue Priority

Solid Queue supports named queues with priority ordering:

```yaml
# config/queue.yml
default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: "real_time"
      threads: 3
      polling_interval: 0.1
    - queues: "default"
      threads: 5
    - queues: "mailers"
      threads: 2
    - queues: "low"
      threads: 1
```

Suggested queue hierarchy:
- `real_time` — anything that feeds the live UI (presence updates, typing indicators)
- `default` — mention notifications, search indexing
- `mailers` — email delivery
- `low` — link unfurling, image processing

---

## Email Jobs

`deliver_later` automatically enqueues via Active Job:

```ruby
NotificationMailer.mention(user, message).deliver_later
NotificationMailer.mention(user, message).deliver_later(wait: 5.minutes)
NotificationMailer.digest(user).deliver_later(wait_until: tomorrow_morning)
```

---

## Retries & Error Handling

```ruby
class UnfurlLinkJob < ApplicationJob
  queue_as :low
  retry_on Net::TimeoutError, wait: :polynomially_longer, attempts: 3
  discard_on ActiveRecord::RecordNotFound

  def perform(message_id, url)
    # fetch URL, extract metadata, save preview
  end
end
```

---

## Monitoring

Solid Queue stores job state in Postgres — query it directly or use Mission Control Jobs, a
Rails engine that provides a web UI for inspecting and retrying jobs.

```ruby
gem "mission_control-jobs"
```

Mount in routes:
```ruby
mount MissionControl::Jobs::Engine, at: "/admin/jobs"
```

---

## References

- [Solid Queue README](https://github.com/rails/solid_queue)
- [Active Job basics](https://guides.rubyonrails.org/active_job_basics.html)
- [Mission Control Jobs](https://github.com/rails/mission_control-jobs)
