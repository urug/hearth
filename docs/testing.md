# Testing Strategy

What to test, how to test it, and how to keep the suite fast.

---

## Test Stack

Rails ships with Minitest. Hearth's Gemfile already includes `capybara` and `selenium-webdriver`
for system tests.

| Layer | Tool |
|---|---|
| Unit tests (models, jobs, mailers) | Minitest (default) |
| Controller / request tests | Rails request specs |
| System tests (full browser) | Capybara + Selenium |
| Action Cable | `ActionCable::Channel::TestCase` |
| Fixtures | Rails fixtures (default) or FactoryBot |

---

## What to Test

### Models — High priority

Business logic lives here. Test it thoroughly.

```ruby
# test/models/message_test.rb
class MessageTest < ActiveSupport::TestCase
  test "body cannot be blank" do
    message = Message.new(body: "", user: users(:alice), room: rooms(:general))
    assert_not message.valid?
    assert_includes message.errors[:body], "can't be blank"
  end

  test "mentions returns users referenced with @" do
    alice = users(:alice)
    message = Message.new(body: "hey @#{alice.display_name}")
    assert_includes message.mentions, alice
  end
end
```

### Jobs — Medium priority

Test that jobs do the right thing, not that they were enqueued:

```ruby
# test/jobs/notify_mentions_job_test.rb
class NotifyMentionsJobTest < ActiveJob::TestCase
  test "sends email to mentioned users" do
    message = messages(:mention_alice)
    assert_emails 1 do
      NotifyMentionsJob.perform_now(message.id)
    end
  end
end
```

### Action Cable Channels — Medium priority

```ruby
# test/channels/room_channel_test.rb
class RoomChannelTest < ActionCable::Channel::TestCase
  test "subscribes to room stream" do
    stub_connection current_user: users(:alice)
    subscribe room_id: rooms(:general).id
    assert subscription.confirmed?
    assert_has_stream_for rooms(:general)
  end
end
```

### System Tests — Selective

System tests are slow. Use them for critical paths only:

- User can sign in and see the message list
- User can post a message and see it appear in real-time
- User receives a notification on mention

```ruby
# test/system/messaging_test.rb
class MessagingTest < ApplicationSystemTestCase
  test "user can send a message" do
    sign_in users(:alice)
    visit room_path(rooms(:general))

    fill_in "message_body", with: "Hello, world!"
    click_button "Send"

    assert_text "Hello, world!"
  end
end
```

### Mailers

```ruby
# test/mailers/notification_mailer_test.rb
class NotificationMailerTest < ActionMailer::TestCase
  test "mention email contains message body" do
    mail = NotificationMailer.mention(users(:bob), messages(:mention_bob))
    assert_equal [users(:bob).email], mail.to
    assert_match messages(:mention_bob).body, mail.body.encoded
  end
end
```

---

## Fixtures vs FactoryBot

Rails fixtures are fast (inserted once per test run, not per test). FactoryBot is more flexible
but slower and can lead to complex factory chains.

**Recommendation:** Start with fixtures. They're underrated and work well for a community project
where contributor familiarity matters.

---

## Action Cable in System Tests

Testing real-time behavior in system tests requires a real Action Cable connection:

```ruby
# test/application_system_test_case.rb
class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :headless_chrome, screen_size: [1400, 1400]
end
```

Use `assert_text` with a wait: Capybara will retry the assertion until the DOM updates or the
timeout is reached (default 2 seconds).

```ruby
# Wait for the broadcast to arrive
assert_text "Hello, world!", wait: 5
```

---

## Test Parallelization

Rails runs tests in parallel by default. Ensure fixtures and database state don't conflict:

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
  parallelize_setup { |worker| ActiveRecord::Base.connection_pool.disconnect! }
end
```

---

## CI Configuration

The `.github/workflows` directory is already present. A basic CI workflow:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/hearth_test
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bin/rails db:test:prepare
      - run: bin/rails test
      - run: bin/rails test:system
```

---

## References

- [Rails testing guide](https://guides.rubyonrails.org/testing.html)
- [Action Cable testing](https://guides.rubyonrails.org/testing.html#testing-action-cable)
- [Capybara docs](https://github.com/teamcapybara/capybara)
- [FactoryBot](https://github.com/thoughtbot/factory_bot_rails) — if fixtures aren't enough
