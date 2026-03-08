---
name: ruby-patterns
description: Idiomatic Ruby patterns, conventions, metaprogramming, blocks/procs/lambdas, Enumerable, concurrency, and gem authoring best practices.
origin: ECC
---

# Ruby Development Patterns

Idiomatic Ruby patterns and best practices for building robust, elegant, and maintainable applications.

## When to Activate

- Writing new Ruby code
- Reviewing Ruby code
- Refactoring existing Ruby code
- Designing Ruby gems or libraries

## Core Principles

### 1. Convention Over Configuration

Ruby and its ecosystem favor sensible defaults and conventions.

```ruby
# frozen_string_literal: true

# Good: Expressive, reads like English
def active_users
  users.select(&:active?)
end

# Bad: Overly verbose
def get_active_users_from_list(user_list)
  result = []
  user_list.each do |u|
    result << u if u.active == true
  end
  result
end
```

### 2. Duck Typing

Ask what an object can do, not what it is.

```ruby
# Good: Duck typing
def process(handler)
  raise ArgumentError, "must respond to #call" unless handler.respond_to?(:call)

  handler.call
end

# Works with any callable
process(-> { puts "lambda" })
process(method(:some_method))
process(MyCallableClass.new)

# Bad: Type checking
def process(handler)
  raise TypeError unless handler.is_a?(Proc)

  handler.call
end
```

### 3. Least Surprise

Code should behave as the reader expects.

```ruby
# Good: Predictable behavior
class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees.to_f
    freeze
  end

  def <=>(other)
    degrees <=> other.degrees
  end

  def to_s
    "#{degrees}°"
  end
end
```

## Blocks, Procs, and Lambdas

### Blocks

```ruby
# Block with do..end for multi-line
users.each do |user|
  send_notification(user)
  log_activity(user)
end

# Block with braces for single-line
names = users.map { |u| u.name }

# Shorthand with Symbol#to_proc
names = users.map(&:name)

# Yielding to blocks
def with_retry(attempts: 3)
  attempts.times do |i|
    return yield
  rescue StandardError => e
    raise if i == attempts - 1

    sleep(2**i)
  end
end

with_retry { api_client.fetch_data }
```

### Procs vs Lambdas

```ruby
# Lambda: strict arity, returns from lambda
validator = ->(value) { value.to_s.length > 3 }
validator.call("test") # => true
validator.("test")     # => true (shorthand)

# Proc: flexible arity, returns from enclosing method
formatter = proc { |name, title| "#{title} #{name}" }
formatter.call("Alice", "Dr.") # => "Dr. Alice"
formatter.call("Alice")        # => " Alice" (no error)

# Method objects
def double(x) = x * 2

[1, 2, 3].map(&method(:double)) # => [2, 4, 6]
```

## Enumerable Patterns

```ruby
# Transformations
users.map(&:name)                          # Extract attribute
users.flat_map(&:orders)                   # Flatten nested collections
users.filter_map { |u| u.name if u.active? } # Map + compact

# Filtering
adults = users.select { |u| u.age >= 18 }
minors = users.reject { |u| u.age >= 18 }
admin = users.find { |u| u.role == :admin }

# Aggregation
total = orders.sum(&:amount)
grouped = items.group_by(&:category)
counts = words.tally  # Ruby 2.7+

# Building collections
lookup = users.each_with_object({}) do |user, hash|
  hash[user.id] = user
end

# Chaining with lazy evaluation
large_file_lines
  .lazy
  .select { |line| line.include?("ERROR") }
  .map(&:strip)
  .first(10)
```

## Immutability Patterns

### Frozen Objects

```ruby
# frozen_string_literal: true

# Freeze constants
VALID_STATUSES = %w[active inactive archived].freeze
CONFIG = { timeout: 30, retries: 3 }.freeze

# Value objects with Data (Ruby 3.2+)
Point = Data.define(:x, :y) do
  def distance_to(other)
    Math.sqrt((x - other.x)**2 + (y - other.y)**2)
  end
end

point = Point.new(x: 3, y: 4)
point.x = 5 # => NoMethodError

# Value objects with frozen Struct
Money = Struct.new(:amount, :currency, keyword_init: true) do
  def initialize(**)
    super
    freeze
  end

  def +(other)
    raise ArgumentError, "currency mismatch" unless currency == other.currency

    self.class.new(amount: amount + other.amount, currency: currency)
  end
end
```

## Service Objects

```ruby
# Plain Ruby class with a single public method
class CreateUser
  def initialize(user_repo:, mailer:)
    @user_repo = user_repo
    @mailer = mailer
  end

  def call(params)
    validate!(params)
    user = @user_repo.create(params)
    @mailer.send_welcome(user)
    user
  end

  private

  def validate!(params)
    raise ArgumentError, "email required" unless params[:email]
    raise ArgumentError, "invalid email" unless params[:email].include?("@")
  end
end

# Usage
service = CreateUser.new(user_repo: UserRepo.new, mailer: Mailer.new)
user = service.call(name: "Alice", email: "alice@example.com")
```

## Module Mixins

```ruby
# Reusable behavior via modules
module Timestamped
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def timestamped_attributes
      @timestamped_attributes ||= []
    end
  end

  def touch
    @updated_at = Time.now
  end
end

# Refinements for scoped monkey-patching
module StringExtensions
  refine String do
    def to_slug
      downcase.gsub(/[^a-z0-9]+/, "-").gsub(/^-|-$/, "")
    end
  end
end

class Post
  using StringExtensions

  def slug
    title.to_slug
  end
end
```

## Error Handling

```ruby
# Custom exception hierarchy
module MyApp
  class Error < StandardError; end
  class ValidationError < Error; end
  class NotFoundError < Error; end

  class AuthenticationError < Error
    attr_reader :code

    def initialize(message, code: 401)
      @code = code
      super(message)
    end
  end
end

# Rescue specific exceptions
def find_user!(id)
  user = repo.find(id)
  raise MyApp::NotFoundError, "User #{id} not found" unless user

  user
rescue ActiveRecord::ConnectionNotEstablished => e
  raise MyApp::Error, "Database unavailable: #{e.message}"
end

# Retry pattern
def with_retry(max_attempts: 3, exceptions: [StandardError])
  attempts = 0
  begin
    attempts += 1
    yield
  rescue *exceptions => e
    raise if attempts >= max_attempts

    sleep(2**attempts * 0.1)
    retry
  end
end
```

## Metaprogramming (Use Sparingly)

```ruby
# Dynamic method definition
class Config
  SETTINGS = %i[host port timeout].freeze

  SETTINGS.each do |setting|
    define_method(setting) do
      @data[setting]
    end

    define_method(:"#{setting}=") do |value|
      @data[setting] = value
    end
  end

  def initialize
    @data = {}
  end
end

# method_missing with respond_to_missing?
class FlexibleConfig
  def initialize(data = {})
    @data = data
  end

  def method_missing(name, *args)
    key = name.to_s.delete_suffix("=").to_sym
    if name.to_s.end_with?("=")
      @data[key] = args.first
    elsif @data.key?(key)
      @data[key]
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    key = name.to_s.delete_suffix("=").to_sym
    @data.key?(key) || name.to_s.end_with?("=") || super
  end
end
```

## Concurrency

### Threads and Fibers

```ruby
# Thread-safe operations with Mutex
class Counter
  def initialize
    @count = 0
    @mutex = Mutex.new
  end

  def increment
    @mutex.synchronize { @count += 1 }
  end

  def value
    @mutex.synchronize { @count }
  end
end

# Ractor for true parallelism (Ruby 3.0+)
results = 4.times.map do |i|
  Ractor.new(i) do |index|
    # Each Ractor runs in parallel
    heavy_computation(index)
  end
end.map(&:take)

# Concurrent gem for thread pools
require "concurrent"

pool = Concurrent::FixedThreadPool.new(5)
futures = urls.map do |url|
  Concurrent::Future.execute(executor: pool) { fetch(url) }
end
results = futures.map(&:value)
```

## Gem Structure

```
my_gem/
├── lib/
│   ├── my_gem.rb              # Entry point
│   └── my_gem/
│       ├── version.rb
│       ├── configuration.rb
│       └── client.rb
├── spec/
│   ├── spec_helper.rb
│   └── my_gem/
│       └── client_spec.rb
├── my_gem.gemspec
├── Gemfile
├── Rakefile
├── LICENSE.txt
└── README.md
```

## Ruby Tooling

```bash
# Linting and formatting
rubocop --autocorrect
rubocop --auto-gen-config          # Generate .rubocop_todo.yml

# Testing
bundle exec rspec
bundle exec rspec --format documentation

# Security
brakeman --no-pager                # Rails security scanner
bundle audit check --update        # Dependency vulnerabilities

# Performance
ruby-prof script.rb                # Profiling
stackprof --text tmp/stackprof.dump # Sampling profiler
```

## Anti-Patterns to Avoid

```ruby
# Bad: rescue Exception (catches signals, syntax errors)
begin
  risky_operation
rescue Exception => e
  # Catches everything including Interrupt, SystemExit
end

# Good: rescue StandardError (default)
begin
  risky_operation
rescue StandardError => e
  logger.error("Operation failed: #{e.message}")
end

# Bad: unless...else
unless user.admin?
  deny_access
else
  grant_access
end

# Good: if...else
if user.admin?
  grant_access
else
  deny_access
end

# Bad: Nested conditionals
def process(user)
  if user
    if user.active?
      if user.verified?
        do_work(user)
      end
    end
  end
end

# Good: Guard clauses
def process(user)
  return unless user
  return unless user.active?
  return unless user.verified?

  do_work(user)
end
```

## Quick Reference

| Idiom | Description |
|-------|-------------|
| `&:method_name` | Symbol#to_proc for simple blocks |
| `frozen_string_literal` | Freeze all string literals in file |
| Guard clauses | Early returns to reduce nesting |
| `Data.define` | Immutable value objects (Ruby 3.2+) |
| `Struct.new` | Simple data containers |
| `each_with_object` | Build collections without mutation |
| `filter_map` | Map + compact in one pass |
| `tally` | Count occurrences |
| `then`/`yield_self` | Pipeline-style transforms |
| Refinements | Scoped monkey-patching |

**Remember**: Ruby optimizes for developer happiness. Write code that is expressive, readable, and follows the principle of least surprise.
