---
paths:
  - "**/*.rb"
  - "**/*.rake"
  - "**/Gemfile"
  - "**/Rakefile"
---
# Ruby Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with Ruby specific content.

## Standards

- Follow the [Ruby community style guide](https://rubystyle.guide/)
- Use `# frozen_string_literal: true` magic comment at the top of every file
- Naming: `snake_case` for methods/variables, `CamelCase` for classes/modules, `SCREAMING_SNAKE_CASE` for constants

## Immutability

Prefer immutable data structures:

```ruby
# frozen_string_literal: true

# Use Struct for simple value objects
Point = Struct.new(:x, :y, keyword_init: true) do
  def initialize(**)
    super
    freeze
  end
end

# Ruby 3.2+ — Data.define for truly immutable value objects
Point = Data.define(:x, :y)

# Freeze collections
VALID_STATUSES = %w[active inactive archived].freeze
```

## Formatting

- **RuboCop** for linting and auto-formatting:
  ```bash
  rubocop --autocorrect
  ```

## Reference

See skill: `ruby-patterns` for comprehensive Ruby idioms and patterns.
