# References

Code, projects, and resources to study when building Hearth.

---

## Reference Applications

### Campfire (37signals / ONCE)
The direct inspiration for Hearth. Built by the team that invented Rails, using the exact stack
we are considering (Hotwire, Action Cable, Solid Cable). Purchase includes full source code.

- Product: https://once.com/campfire
- 37signals engineering blog: https://dev.37signals.com
- GitHub org (open source Rails tooling): https://github.com/37signals

Worth studying for: how they structure channels, broadcasting, presence, and message threading.
The source reflects years of production experience with this exact problem.

---

### Basecamp
The original 37signals product. Long history of pioneering server-rendered real-time patterns.
Not open source, but the engineering blog documents many architectural decisions.

- Engineering blog: https://dev.37signals.com
- Notable posts: search for "Hotwire", "Turbo", "Action Cable" on the blog

---

### Bullet Train
Open source Rails SaaS starter kit. Not a chat app, but a well-structured Rails 7/8 codebase
with authentication, teams, permissions, and Hotwire patterns throughout. Good reference for
how to structure a production Rails app.

- https://bullettrain.co
- https://github.com/bullet-train-co/bullet-train-core

---

### Forem (dev.to)
Open source Rails app powering dev.to. Large, production Rails codebase with real-time
notifications, feeds, and chat-adjacent features. Useful for seeing how a real Rails app
handles scale, background jobs, and complex models.

- https://github.com/forem/forem

---

## Key Gems (Already in Hearth or Likely Additions)

### Solid Cable
Postgres-backed Action Cable adapter. Eliminates Redis. Already in Hearth's Gemfile.

- https://github.com/rails/solid_cable

### Turbo Rails
The Hotwire Turbo integration for Rails. Provides `broadcast_append_to`, `turbo_stream_from`,
and all the broadcasting helpers.

- https://github.com/hotwired/turbo-rails
- https://turbo.hotwired.dev

### Stimulus
The JS framework for sprinkling behavior onto server-rendered HTML.

- https://stimulus.hotwired.dev
- https://github.com/hotwired/stimulus

### Action Text + Trix
Rails' built-in rich text solution. Trix is the editor; Action Text handles storage, attachments,
and rendering. Natural fit for a message composer.

- https://guides.rubyonrails.org/action_text_overview.html
- https://trix-editor.org

### Active Storage
File attachment handling built into Rails. Handles direct uploads, variants, and cloud storage
(S3, GCS, Azure). Required for message file attachments.

- https://guides.rubyonrails.org/active_storage_overview.html

### Devise
The standard Rails authentication gem. Handles registration, login, password reset, email
confirmation, and more.

- https://github.com/heartcombo/devise

### Pundit / Action Policy
Authorization gems for controlling who can read/write which channels and messages.

- Pundit: https://github.com/varvet/pundit
- Action Policy: https://actionpolicy.evilmartians.io (more performant, Rails-native feel)

### Pagy
Lightweight, fast pagination. Relevant for loading older message history.

- https://github.com/ddnexus/pagy

---

## Tutorials & Guides

### GoRails — Action Cable Chat
Comprehensive screencast series on building a chat app with Rails and Action Cable. Covers
channels, private messaging, and real-time updates.

- https://gorails.com (search "Action Cable chat")

### Evil Martians — HTMX in Rails
Detailed walkthrough of using HTMX with Rails as an alternative to Hotwire.

- https://evilmartians.com/chronicles/htmx-in-rails

### Hotwire Handbook
The official Turbo documentation. Essential reading if going the Hotwire route.

- https://turbo.hotwired.dev/handbook/introduction

### The Hypermedia Systems Book
Free online book written by the HTMX author. Covers the philosophy behind server-driven HTML,
REST, and hypermedia. Useful context regardless of which architecture is chosen.

- https://hypermedia.systems

### Rails Guides — Action Cable
Official documentation covering the full Action Cable API, channels, broadcasting, and
connection authentication.

- https://guides.rubyonrails.org/action_cable_overview.html

---

## Open Source Chat Apps (Non-Rails, for Comparison)

### Zulip
Open source team chat with a threading model different from Slack (topic-based threads inside
streams). Python/Django backend, actively maintained.

- https://github.com/zulip/zulip

### Mattermost
Open source Slack alternative. Go backend, React frontend. Full-featured. Worth studying for
feature scope and UX patterns.

- https://github.com/mattermost/mattermost

### Rocket.Chat
Open source, self-hosted team chat. Node.js backend. Large feature set including calls, bots,
and federation.

- https://github.com/RocketChat/Rocket.Chat

---

## Design References

### Slack
The primary UX reference. Note what works (channel sidebar, message threading, reactions,
keyboard shortcuts) and what to cut for v1.

- https://slack.com

### Basecamp Campfire (ONCE)
Simpler than Slack. Direct inspiration for Hearth's scope.

- https://once.com/campfire
