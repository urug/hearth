# Architecture: Phlex

## Overview

Phlex is a Ruby gem that replaces ERB (and Slim/Haml) with plain Ruby objects for rendering
HTML. Instead of templates, you write view components as Ruby classes with a `view_template`
method. The output is the same HTML — it's a rendering layer, not a real-time architecture.

Phlex pairs with any of the other architectures (Hotwire, HTMX, Inertia) for the view layer.
It is documented here separately because it meaningfully changes how the app is structured and
how contributors interact with the codebase.

## How It Works

```ruby
# Instead of app/views/messages/_message.html.erb:
<div class="message">
  <span class="author"><%= message.user.name %></span>
  <p><%= message.body %></p>
</div>

# You write app/views/messages/message_component.rb:
class Messages::MessageComponent < Phlex::HTML
  def initialize(message:)
    @message = message
  end

  def view_template
    div(class: "message") do
      span(class: "author") { @message.user.name }
      p { @message.body }
    end
  end
end
```

Rendering from a controller or another component:
```ruby
render Messages::MessageComponent.new(message: @message)
```

## Stack

Phlex is a gem, not a framework. It slots into an existing Rails stack:

| Layer | Technology |
|-------|-----------|
| View rendering | [Phlex](https://www.phlex.fun) |
| Real-time | Whatever the rest of the stack uses (Turbo Streams, HTMX SSE, etc.) |
| Asset pipeline | Unchanged |

Phlex works alongside ERB — you can migrate incrementally or use it for new components only.

## Pros

- **Pure Ruby** — Views are Ruby objects. You get autocomplete, `go to definition`, refactoring
  tools, and test coverage on your view layer for free.
- **Testable** — Components can be unit tested without rendering a full request.
- **Composable** — Components are just objects; composition is natural Ruby (`include`, inheritance,
  delegation).
- **No template language to learn** — New contributors who know Ruby don't need to learn ERB
  idioms or template scoping rules.
- **Works with Turbo** — `broadcast_append_to` accepts a Phlex component as the partial.
- **Performance** — Phlex renders faster than ERB in benchmarks (direct string building vs.
  template compilation).

## Cons

- **Not Rails default** — Departs from conventions most Rails developers know. ERB is universal;
  Phlex is not.
- **HTML in Ruby feels odd at first** — The syntax is clean but takes adjustment, especially for
  contributors coming from template-heavy backgrounds.
- **Tooling is younger** — HTML linters, formatters, and editor plugins don't understand Phlex
  the way they understand ERB.
- **Still maturing** — The API has had breaking changes between versions; ecosystem (Rails
  integrations, generators) is still catching up.
- **Not a solution to real-time** — Phlex alone does nothing for chat's core challenge. It must
  be paired with a real-time architecture.

## Fit for Hearth

Interesting optional layer. Phlex wouldn't change how real-time works — it changes how views are
written. For a Ruby group, the appeal is clear: everything in Ruby, views are testable objects,
no context switching into template syntax.

The risk is that it's one more non-default thing for contributors to learn. Worth discussing as
an option, not necessarily a default choice.

## Phlex + Turbo Streams

Phlex components work as Turbo Stream targets:

```ruby
# Broadcasting a Phlex component
after_create_commit -> {
  broadcast_append_to(
    channel,
    target: "messages",
    html: render(Messages::MessageComponent.new(message: self))
  )
}
```

## References

- [Phlex docs](https://www.phlex.fun)
- [phlex-rails gem](https://github.com/phlex-ruby/phlex-rails)
- [Phlex vs ViewComponent comparison](https://www.phlex.fun/vs/view-component)
- [ViewComponent](https://viewcomponent.org) — GitHub's component library for Rails; similar idea,
  different approach, more mature
