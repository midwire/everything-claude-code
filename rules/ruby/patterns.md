---
paths:
  - "**/*.rb"
  - "**/*.rake"
  - "**/Gemfile"
  - "**/Rakefile"
---
# Ruby Patterns

> This file extends [common/patterns.md](../common/patterns.md) with Ruby specific content.

## Duck Typing

```ruby
# Use respond_to? instead of type checking
def process(handler)
  raise ArgumentError, "must respond to #call" unless handler.respond_to?(:call)

  handler.call
end
```

## Service Objects

```ruby
class CreateUser
  def initialize(params)
    @params = params
  end

  def call
    validate!
    user = User.new(@params)
    user.save!
    user
  end

  private

  def validate!
    raise ArgumentError, "email required" unless @params[:email]
  end
end
```

## Value Objects

```ruby
# Ruby 3.2+
UserEmail = Data.define(:address) do
  def initialize(address:)
    raise ArgumentError, "invalid email" unless address.include?("@")

    super
  end
end

# Pre-3.2 — frozen Struct
Money = Struct.new(:amount, :currency, keyword_init: true) do
  def initialize(**)
    super
    freeze
  end
end
```

## Module Mixins

```ruby
module Auditable
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def audited_fields(*fields)
      @audited_fields = fields
    end
  end
end
```

## Enumerable Patterns

```ruby
# Prefer Enumerable methods over manual iteration
names = users.map(&:name)
adults = users.select { |u| u.age >= 18 }
total = orders.sum(&:amount)
grouped = items.group_by(&:category)
```
