# Message Formatting

How message bodies are stored, rendered, and displayed.

---

## Options

### Option 1: Plain Text (v1 Starting Point)

Store raw text, display as-is. Simple, safe, no dependencies.

Downside: no code blocks, no bold/italic, URLs not clickable unless auto-linked.

### Option 2: Markdown

Store Markdown source, render to HTML on the server before display.

```ruby
gem "redcarpet"   # or
gem "kramdown"    # or
gem "commonmarker" # CommonMark spec, C extension, fastest
```

```ruby
# app/helpers/messages_helper.rb
def render_message(body)
  renderer = Redcarpet::Render::HTML.new(
    filter_html: true,
    no_images: false,
    hard_wrap: true
  )
  markdown = Redcarpet::Markdown.new(renderer,
    autolink: true,
    fenced_code_blocks: true,
    strikethrough: true
  )
  markdown.render(body).html_safe
end
```

**Important:** Always sanitize HTML output with `rails_html_sanitizer` or `sanitize` helper
to prevent XSS from user-supplied content.

### Option 3: Action Text + Trix

Rails' built-in rich text solution. Trix is the WYSIWYG editor; Action Text handles storage
and rendering. Content is stored as HTML internally.

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  has_rich_text :body
end
```

```erb
<%= form.rich_text_area :body %>
```

**Pros:** First-party Rails solution, handles attachments natively, good UX.
**Cons:** Trix is opinionated (no Markdown input), larger payload, harder to customize editor behavior.

### Option 4: Custom Subset (Slack-style)

Define a limited syntax (bold with `*`, italic with `_`, code with backticks) and parse it
server-side. Familiar to users, full control over the feature set.

Libraries to consider:
- Write a small parser with `strscan` (Ruby stdlib)
- Use `commonmarker` but disable most features

---

## Recommendation

**Start with Markdown via `commonmarker`** — it's fast (C extension), spec-compliant, and
familiar. Add syntax highlighting separately. Avoid Action Text unless the WYSIWYG editor is
specifically desired.

---

## Syntax Highlighting

Highlight code blocks in messages:

### Server-side (recommended)

```ruby
gem "rouge"  # pure Ruby, no JS needed
```

```ruby
# In your Markdown renderer, override block_code:
class HTMLWithRouge < Redcarpet::Render::HTML
  def block_code(code, language)
    formatter = Rouge::Formatters::HTML.new
    lexer = Rouge::Lexer.find_fancy(language, code) || Rouge::Lexers::PlainText
    "<pre><code>#{formatter.format(lexer.lex(code))}</code></pre>"
  end
end
```

Include a Rouge CSS theme in your stylesheet — no JS required.

### Client-side (alternative)

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/..."></script>
```

Runs in the browser after the page loads. More flexible language support, slightly delayed render.

---

## Link Previews (Unfurling)

Fetch metadata for pasted URLs and display a preview card.

```ruby
# app/jobs/unfurl_url_job.rb
class UnfurlUrlJob < ApplicationJob
  queue_as :low

  def perform(message_id)
    message = Message.find(message_id)
    urls = URI.extract(message.body, ["http", "https"])

    urls.first(3).each do |url|
      meta = fetch_opengraph(url)
      message.link_previews.create!(url: url, **meta) if meta
    end

    # Broadcast updated message partial to connected clients
    message.broadcast_replace_to(message.room)
  end
end
```

Libraries:
- [`open_graph_reader`](https://github.com/Drenmi/open_graph_reader) — parse OpenGraph tags
- [`link_preview`](https://github.com/gottfrois/link_preview) — full unfurling gem
- Roll your own with `Nokogiri` + `open-uri`

---

## Sanitization

Always sanitize user-provided HTML before rendering:

```ruby
# config/initializers/content_security.rb
ALLOWED_TAGS = %w[p br strong em code pre blockquote ul ol li a h1 h2 h3 mark]
ALLOWED_ATTRS = { "a" => ["href", "title"], "code" => ["class"] }

def safe_render(html)
  ActionController::Base.helpers.sanitize(html,
    tags: ALLOWED_TAGS,
    attributes: ALLOWED_ATTRS
  )
end
```

---

## References

- [commonmarker gem](https://github.com/gjtorikian/commonmarker)
- [Redcarpet gem](https://github.com/vmg/redcarpet)
- [Rouge syntax highlighter](https://github.com/rouge-ruby/rouge)
- [Action Text overview](https://guides.rubyonrails.org/action_text_overview.html)
- [Rails sanitize helper](https://api.rubyonrails.org/classes/ActionView/Helpers/SanitizeHelper.html)
