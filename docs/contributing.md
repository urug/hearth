# Contributing

How to get Hearth running locally and start contributing.

---

## Prerequisites

| Tool | Version | Install |
|---|---|---|
| Ruby | 3.4.x | `rbenv install 3.4.9` or via mise |
| PostgreSQL | 14+ | Homebrew: `brew install postgresql` |
| Node.js | Not required | (import maps, no build step) |

---

## Setup

```bash
# Clone
git clone https://github.com/your-org/hearth.git
cd hearth

# Install gems
bundle install

# Database setup
bin/rails db:create db:migrate db:seed

# Start the server
bin/dev
```

Visit `http://localhost:3000`.

---

## bin/dev

`bin/dev` uses Foreman (or `overmind`) to run multiple processes:

```
# Procfile.dev
web: bin/rails server
```

If you add background jobs in development:
```
web:    bin/rails server
worker: bin/rails solid_queue:start
```

---

## Running Tests

```bash
# All tests
bin/rails test

# System tests (requires Chrome)
bin/rails test:system

# Single file
bin/rails test test/models/message_test.rb

# Single test by line
bin/rails test test/models/message_test.rb:15
```

---

## Code Style

Hearth uses RuboCop with the Rails Omakase config (already in `.rubocop.yml`):

```bash
bundle exec rubocop          # check
bundle exec rubocop -a       # auto-fix safe offenses
```

CI runs RuboCop automatically. Fix offenses before opening a PR.

---

## Security Checks

```bash
bundle exec brakeman         # static analysis for security vulnerabilities
bundle exec bundler-audit    # check gems for known CVEs
```

Both run in CI. Brakeman warnings must be addressed or explicitly ignored with a comment.

---

## Pull Request Guidelines

- Keep PRs focused — one feature or fix per PR
- Add tests for new behavior
- Run the full test suite before opening a PR: `bin/rails test test:system`
- Write clear commit messages: what changed and why, not just what

---

## Project Structure

```
app/
  channels/      # Action Cable channels
  controllers/   # Request handling
  jobs/          # Background jobs
  mailers/       # Email templates
  models/        # Business logic + database
  views/         # ERB templates

config/
  routes.rb      # URL routing
  cable.yml      # Action Cable config
  queue.yml      # Solid Queue config

db/
  migrate/       # Database migrations
  seeds.rb       # Development seed data

docs/            # Architecture decisions and research (you are here)
test/            # Minitest suite
```

---

## Seed Data

`db/seeds.rb` should create enough data to develop against:
- A few users (with predictable passwords for development)
- Several rooms including `#general`
- A backlog of messages

```bash
bin/rails db:seed       # run seeds
bin/rails db:reset      # drop, recreate, migrate, seed
```

---

## Useful Commands

```bash
bin/rails console                    # REPL with full Rails environment
bin/rails routes                     # list all routes
bin/rails db:migrate                 # run pending migrations
bin/rails db:rollback                # undo last migration
bin/rails generate model Foo         # generate a model
bin/rails generate migration AddX    # generate a migration
```

---

## Getting Help

- Open an issue on GitHub
- Ask in the `#hearth` channel (once it exists)
- Bring questions to the URUG meetup
