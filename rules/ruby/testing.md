---
paths:
  - "**/*.rb"
  - "**/*.rake"
  - "**/Gemfile"
  - "**/Rakefile"
---
# Ruby Testing

> This file extends [common/testing.md](../common/testing.md) with Ruby specific content.

## Framework

Use **RSpec** as the primary testing framework (Minitest as alternative).

## Coverage

```bash
# Add simplecov to Gemfile
gem "simplecov", require: false, group: :test

# In spec/spec_helper.rb
require "simplecov"
SimpleCov.start
```

## Test Organization

Use `describe`/`context`/`it` blocks:

```ruby
RSpec.describe CreateUser do
  describe "#call" do
    context "with valid params" do
      it "creates a user" do
        result = described_class.new(name: "Alice", email: "a@b.com").call
        expect(result).to be_persisted
      end
    end

    context "with missing email" do
      it "raises an error" do
        expect { described_class.new(name: "Alice").call }
          .to raise_error(ArgumentError)
      end
    end
  end
end
```

## Factories

Prefer **factory_bot** over fixtures:

```ruby
FactoryBot.define do
  factory :user do
    name { "Alice" }
    email { "alice@example.com" }
  end
end
```

## Mocking

Use built-in RSpec mocks:

```ruby
allow(service).to receive(:call).and_return(result)
expect(notifier).to have_received(:notify).with(user)
```

## Reference

See skill: `ruby-testing` for detailed RSpec patterns, shared examples, and VCR usage.
See skill: `rails-tdd` for Rails-specific testing with request specs and Capybara.
