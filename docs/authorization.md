# Authorization

Who can access what, and how to enforce it.

---

## Core Questions

- Can this user read messages in this room?
- Can this user post to this room?
- Can this user edit or delete this message?
- Can this user manage (archive, rename) this room?
- Can this user access this DM conversation?

---

## Options

### Option 1: Pundit (Recommended for v1)

Pundit is a minimal authorization gem based on plain Ruby policy objects.

```ruby
gem "pundit"
```

Each model gets a policy class:

```ruby
# app/policies/message_policy.rb
class MessagePolicy < ApplicationPolicy
  def create?
    user.member_of?(record.room)
  end

  def destroy?
    record.user == user || user.admin_of?(record.room)
  end
end
```

In controllers:
```ruby
def create
  @message = @room.messages.build(message_params.merge(user: Current.user))
  authorize @message
  # ...
end
```

**Pros:** Simple, readable, easy to test, no DSL to learn.
**Cons:** Manual scoping; requires discipline to call `authorize` everywhere.

### Option 2: Action Policy

A more performant, Rails-native alternative to Pundit. Supports policy caching (important for
chat where authorization is checked on every message render), i18n, and `pre_check` hooks.

```ruby
gem "action_policy"
```

```ruby
class MessagePolicy < ApplicationPolicy
  def create?
    user.member_of?(record.room)
  end

  relation_scope(:default) do |relation|
    relation.where(room: user.rooms)
  end
end
```

**Pros:** Faster than Pundit (memoizes policy lookups), better Rails integration, scopes built in.
**Cons:** Slightly more to learn than Pundit.

### Option 3: Inline (No gem)

For very simple cases, authorization can live directly in controllers and models:

```ruby
before_action :require_membership

def require_membership
  redirect_to root_path unless Current.user.member_of?(@room)
end
```

Fine for v1 if authorization rules are simple. Gets messy quickly.

---

## Recommendation

**Pundit** for v1 — straightforward, testable, no magic. Upgrade to **Action Policy** if policy
lookup performance becomes a concern (it will matter when rendering large message lists with
per-message authorization checks).

---

## Key Authorization Rules for Hearth

| Action | Rule |
|---|---|
| View room | User has a Membership for that room |
| Post message | User has a Membership for that room |
| Edit message | Message author only |
| Delete message | Message author or room admin |
| Archive room | Room admin or app admin |
| View DM conversation | User is a Participant |
| Add member to room | Room admin |

---

## Room Membership Enforcement

Always scope room queries through the user's memberships — never query rooms directly:

```ruby
# Bad — exposes all rooms
Room.find(params[:id])

# Good — scoped to current user
Current.user.rooms.find(params[:id])
```

This prevents unauthorized access even if authorization checks are missed.

---

## References

- [Pundit gem](https://github.com/varvet/pundit)
- [Action Policy gem](https://actionpolicy.evilmartians.io)
- [Rails security guide](https://guides.rubyonrails.org/security.html)
