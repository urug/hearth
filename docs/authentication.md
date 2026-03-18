# Authentication

How users register, log in, and stay logged in.

---

## Options

### Option 1: Rails 8 Built-in Auth Generator (Recommended for v1)

Rails 8 ships with a first-party authentication generator:

```bash
bin/rails generate authentication
```

This generates:
- `User` model with `password_digest` (`has_secure_password`)
- `Session` model (database-backed sessions — not cookies)
- `SessionsController` (login/logout)
- `Authentication` concern (included in `ApplicationController`)
- Password reset via signed tokens
- `Current.user` via `CurrentAttributes`

No gem required. Minimal magic. Easy to read and extend.

**What it does not include:** email confirmation, OAuth, remember me tokens, account lockout.
These can be added manually or via gems.

### Option 2: Devise

The standard Rails authentication gem. Includes everything out of the box:

```ruby
gem "devise"
```

Modules: `database_authenticatable`, `registerable`, `recoverable`, `rememberable`,
`validatable`, `confirmable`, `lockable`, `omniauthable`.

**Pros:** Fully featured, well-documented, enormous community.
**Cons:** Heavy, lots of magic, harder to customize, views need overriding.

### Option 3: Devise + OmniAuth

Add OAuth (Google, GitHub, Slack) on top of Devise:

```ruby
gem "devise"
gem "omniauth-google-oauth2"
gem "omniauth-github"
```

For a Ruby users group, GitHub OAuth is a natural fit — everyone has a GitHub account.

---

## Recommendation

Start with the **Rails 8 auth generator**. It produces readable, ownable code. If requirements
grow (OAuth, email confirmation, lockout), add Devise later or bolt on OmniAuth directly.

GitHub OAuth (`omniauth-github`) is worth adding early — it eliminates password management and
URUG members all have GitHub accounts.

---

## Session Security Considerations

- Sessions are database-backed in Rails 8 auth (vs. cookie-based) — easier to invalidate
- Set `config.force_ssl = true` in production
- Use `has_secure_password` — bcrypt by default, secure against timing attacks
- Set session expiry: Rails 8 auth stores sessions in a `sessions` table via the generated `Session` model. Add an `expires_at` column and scope queries to non-expired sessions rather than using `session_store` config.

---

## Current User Pattern

Rails 8 auth generates a `Current` object via `ActiveSupport::CurrentAttributes`:

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end
```

Available everywhere as `Current.user`. No need to pass user through controller/view chain.

---

## References

- [Rails 8 auth generator PR](https://github.com/rails/rails/pull/50446)
- [has_secure_password docs](https://api.rubyonrails.org/classes/ActiveModel/SecurePassword/ClassMethods.html)
- [Devise gem](https://github.com/heartcombo/devise)
- [omniauth-github](https://github.com/omniauth/omniauth-github)
