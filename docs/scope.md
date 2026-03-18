# Scope

What Hearth is, what it isn't, and what we're building first.

---

## What Hearth Is

A self-hosted group chat for URUG. Persistent message history, real-time messaging,
organized into rooms. Owned and operated by the group — no vendor, no data loss, no
90-day message cap.

---

## What Hearth Is Not

- A Slack replacement for large organizations
- A mobile app (web only for v1)
- A public-facing platform (invite-only, known members)
- A feature-complete product on day one

---

## v1 — Get People Talking

The minimum needed for the group to actually use it.

| Feature | Notes |
|---|---|
| User accounts | Register, log in, set display name |
| Rooms | `#general`, `#random`, `#jobs` to start |
| Real-time messaging | Post a message, everyone sees it immediately |
| Persistent history | All messages kept indefinitely |
| Mentions | `@username` highlights and notifies |
| Basic formatting | Code blocks, bold, italic — enough for a dev group |

**Not in v1:** DMs, threads, reactions, file uploads, search, notifications.

---

## v2 — Make It Useful

Features that make the app worth using daily.

| Feature | Notes |
|---|---|
| Direct messages | 1:1 private conversations |
| Reactions | Emoji responses to messages |
| File uploads | Images and documents |
| Search | Find messages by content |
| Email notifications | Mentions and DMs send email |
| Message editing | Fix typos after sending |

---

## v3 — Polish

Features that make it feel finished.

| Feature | Notes |
|---|---|
| Threads | Replies scoped to a message |
| Link previews | Unfurl pasted URLs |
| Presence | Show who is online |
| Typing indicators | "Alice is typing..." |
| Pinned messages | Highlight important room content |
| Invite links | Shareable signup URLs |

---

## Out of Scope (For Now)

These are good ideas that we are explicitly deferring to avoid scope creep:

- Native mobile app
- Video/audio calls
- Bots / integrations
- Federation / multi-server
- Public channels / open registration
- Polls, events, RSVP
- End-to-end encryption

---

## Definition of Done for v1

Hearth v1 is done when:

1. A new member can register and log in
2. They can read the history of `#general`
3. They can post a message and others see it in real-time
4. They can be mentioned and receive a notification
5. It is deployed and accessible at a real URL
