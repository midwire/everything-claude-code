---
paths:
  - "**/*.rb"
  - "**/*.rake"
  - "**/Gemfile"
  - "**/Rakefile"
---
# Ruby Hooks

> This file extends [common/hooks.md](../common/hooks.md) with Ruby specific content.

## PostToolUse Hooks

Configure in `~/.claude/settings.json`:

- **RuboCop**: Auto-format `.rb` files after edit
- **Brakeman**: Run security scan after editing Rails controllers/models (Rails projects only)

## Warnings

- Warn about `puts`/`p`/`pp` statements in edited files (use `Logger` for plain Ruby, or `Rails.logger` in Rails projects)
