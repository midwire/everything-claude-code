---
paths:
  - "**/*.rb"
  - "**/*.rake"
  - "**/Gemfile"
  - "**/Rakefile"
---
# Ruby Security

> This file extends [common/security.md](../common/security.md) with Ruby specific content.

## Secret Management

```ruby
# Use ENV.fetch — raises KeyError if missing
api_key = ENV.fetch("API_KEY")

# With a default (only for non-sensitive values)
port = ENV.fetch("PORT", 3000)
```

## Security Scanning

- **Brakeman** for Rails static analysis:
  ```bash
  brakeman --no-pager
  ```
- **bundler-audit** for dependency vulnerabilities:
  ```bash
  bundle audit check --update
  ```

## Dangerous Methods

- Use `YAML.safe_load` instead of `YAML.load`
- Avoid `eval`, `instance_eval`, `class_eval` with user input
- Avoid `send`/`public_send` with user-controlled method names
- Use `Shellwords.shellescape` when passing user input to shell commands

## Reference

See skill: `rails-security` for Rails-specific security guidelines (if applicable).
