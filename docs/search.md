# Search

Full-text search across message history.

---

## Options

### Option 1: pg_search (Recommended for v1)

Postgres has built-in full-text search via `tsvector`/`tsquery`. The `pg_search` gem wraps it
in a clean ActiveRecord interface with no additional infrastructure.

```ruby
gem "pg_search"
```

```ruby
class Message < ApplicationRecord
  include PgSearch::Model

  pg_search_scope :search_body,
    against: :body,
    using: {
      tsearch: {
        prefix: true,          # "hel" matches "hello"
        highlight: {           # returns highlighted snippets
          StartSel: "<mark>",
          StopSel: "</mark>"
        }
      }
    }
end
```

Query:
```ruby
Message
  .joins(:room)
  .merge(Current.user.rooms)   # only rooms the user can see
  .search_body("action cable")
  .order(created_at: :desc)
  .limit(20)
```

**Pros:** No extra infrastructure, already in Postgres, good enough for thousands of messages.
**Cons:** Not as fast or feature-rich as dedicated search engines at large scale; no fuzzy
matching; relevance ranking is basic.

### Option 2: Meilisearch

A fast, typo-tolerant search engine. Free to self-host; managed cloud option available.

```ruby
gem "meilisearch-rails"
```

```ruby
class Message < ApplicationRecord
  include MeiliSearch::Rails

  meilisearch do
    attribute :body, :created_at, :room_id
    filterable_attributes [:room_id]
    searchable_attributes [:body]
  end
end
```

**Pros:** Typo-tolerant, fast, great relevance ranking, easy setup.
**Cons:** Separate process to run and maintain; requires syncing index on message create/update/delete.

### Option 3: Elasticsearch / OpenSearch

Enterprise-grade search. Powerful but operationally heavy.

```ruby
gem "elasticsearch-model"
```

**Pros:** Battle-tested at scale, rich query DSL, aggregations.
**Cons:** High memory usage (~1GB+), complex ops, significant overkill for Hearth v1.

---

## Recommendation

**pg_search for v1.** It requires zero new infrastructure and is sufficient for a community
chat app. Migrate to Meilisearch when search quality or performance becomes a pain point.

---

## Adding a Postgres Search Index

For performance, add a `tsvector` column and GIN index:

```ruby
# migration
add_column :messages, :body_search, :tsvector
add_index :messages, :body_search, using: :gin

# Keep it updated automatically with a Postgres trigger (or via after_save callback)
```

Or use pg_search's multimodel search to search across messages and rooms simultaneously.

---

## Scoping Search to Authorized Rooms

Always scope search results to rooms the user is a member of:

```ruby
def search(query, user)
  Message
    .joins(:room)
    .where(rooms: { id: user.room_ids })
    .search_body(query)
    .includes(:user, :room)
    .order(created_at: :desc)
    .limit(50)
end
```

Never return messages from rooms the user isn't a member of, even if the query matches.

---

## References

- [pg_search gem](https://github.com/Casecommons/pg_search)
- [Meilisearch Rails](https://github.com/meilisearch/meilisearch-rails)
- [Postgres full-text search docs](https://www.postgresql.org/docs/current/textsearch.html)
