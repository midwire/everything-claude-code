---
name: ruby-testing
description: Ruby testing strategies using RSpec, TDD methodology, factories, mocking, shared examples, and coverage requirements.
origin: ECC
---

# Ruby Testing Patterns

Comprehensive testing strategies for Ruby applications using RSpec, TDD methodology, and best practices.

## When to Activate

- Writing new Ruby code (follow TDD: red, green, refactor)
- Designing test suites for Ruby projects
- Reviewing Ruby test coverage
- Setting up testing infrastructure

## TDD Workflow

### Red-Green-Refactor Cycle

```ruby
# Step 1: RED - Write failing test
RSpec.describe Calculator do
  describe "#add" do
    it "returns the sum of two numbers" do
      calculator = Calculator.new
      expect(calculator.add(2, 3)).to eq(5)
    end
  end
end

# Step 2: GREEN - Write minimal implementation
class Calculator
  def add(a, b)
    a + b
  end
end

# Step 3: REFACTOR if needed
```

### Coverage Requirements

- **Target**: 80%+ code coverage
- **Critical paths**: 100% coverage required
- Use `simplecov` to measure coverage

```ruby
# spec/spec_helper.rb
require "simplecov"
SimpleCov.start do
  add_filter "/spec/"
  minimum_coverage 80
end
```

## RSpec Fundamentals

### Test Organization

```ruby
RSpec.describe UserService do
  describe "#create" do
    context "with valid params" do
      it "creates a user" do
        result = described_class.new.create(name: "Alice", email: "a@b.com")
        expect(result).to be_a(User)
        expect(result.name).to eq("Alice")
      end

      it "sends a welcome email" do
        expect { described_class.new.create(valid_params) }
          .to change { ActionMailer::Base.deliveries.count }.by(1)
      end
    end

    context "with missing email" do
      it "raises a validation error" do
        expect { described_class.new.create(name: "Alice") }
          .to raise_error(ArgumentError, /email required/)
      end
    end

    context "when database is unavailable" do
      before { allow(UserRepo).to receive(:create).and_raise(ConnectionError) }

      it "wraps the error" do
        expect { described_class.new.create(valid_params) }
          .to raise_error(ServiceError, /database unavailable/i)
      end
    end
  end
end
```

### Matchers

```ruby
# Equality
expect(result).to eq(expected)
expect(result).to eql(expected)     # type + value
expect(result).to equal(expected)   # object identity

# Truthiness
expect(result).to be true
expect(result).to be_truthy
expect(result).to be_falsey
expect(result).to be_nil

# Comparisons
expect(result).to be > 0
expect(result).to be_between(1, 10)

# Collections
expect(array).to include(item)
expect(array).to contain_exactly(1, 2, 3)  # order-independent
expect(array).to match_array([3, 1, 2])
expect(hash).to include(key: value)

# Types
expect(result).to be_a(String)
expect(result).to respond_to(:call)

# Changes
expect { action }.to change { value }.from(0).to(1)
expect { action }.to change { value }.by(1)

# Exceptions
expect { action }.to raise_error(ArgumentError)
expect { action }.to raise_error(ArgumentError, /message/)

# Output
expect { action }.to output("hello\n").to_stdout
```

## Factories with FactoryBot

### Defining Factories

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { "Alice" }
    sequence(:email) { |n| "user#{n}@example.com" }
    role { :member }
    active { true }

    trait :admin do
      role { :admin }
    end

    trait :inactive do
      active { false }
    end

    trait :with_orders do
      transient do
        order_count { 3 }
      end

      after(:create) do |user, evaluator|
        create_list(:order, evaluator.order_count, user: user)
      end
    end

    factory :admin_user, traits: [:admin]
  end
end
```

### Using Factories

```ruby
# Build (in memory, not saved)
user = build(:user)
user = build(:user, name: "Bob")

# Create (saved to database)
user = create(:user)
user = create(:user, :admin)
user = create(:user, :with_orders, order_count: 5)

# Build list
users = build_list(:user, 3)

# Attributes hash
attrs = attributes_for(:user)
```

## Mocking and Stubbing

### RSpec Mocks

```ruby
# Stubs
allow(service).to receive(:call).and_return(result)
allow(service).to receive(:call).with("arg").and_return(result)
allow(service).to receive(:call).and_raise(StandardError)

# Message expectations
expect(mailer).to receive(:send_welcome).with(user).once
expect(logger).to have_received(:info).with(/created/)

# Spy pattern (verify after the fact)
mailer = instance_spy(Mailer)
service = UserService.new(mailer: mailer)
service.create(params)
expect(mailer).to have_received(:send_welcome)

# Doubles
api_client = instance_double(ApiClient, fetch: response)
allow(api_client).to receive(:fetch).and_return(data)

# Partial doubles (real object, stubbed method)
allow(Time).to receive(:now).and_return(frozen_time)
```

### Stubbing Chains

```ruby
# Stub method chain
allow(User).to receive_message_chain(:active, :verified, :count).and_return(42)
```

## Shared Examples and Contexts

### Shared Examples

```ruby
# spec/support/shared_examples/auditable.rb
RSpec.shared_examples "auditable" do
  it "tracks created_at" do
    expect(subject).to respond_to(:created_at)
  end

  it "tracks updated_at" do
    expect(subject).to respond_to(:updated_at)
  end
end

# Usage
RSpec.describe User do
  it_behaves_like "auditable"
end

RSpec.describe Order do
  it_behaves_like "auditable"
end
```

### Shared Contexts

```ruby
RSpec.shared_context "authenticated user" do
  let(:current_user) { create(:user) }

  before do
    allow(controller).to receive(:current_user).and_return(current_user)
  end
end

RSpec.describe OrdersController do
  include_context "authenticated user"

  describe "GET #index" do
    it "returns user orders" do
      order = create(:order, user: current_user)
      get :index
      expect(response.parsed_body).to include(order.as_json)
    end
  end
end
```

## Let and Subject

```ruby
RSpec.describe Calculator do
  subject(:calculator) { described_class.new }

  let(:a) { 2 }
  let(:b) { 3 }

  # let is lazy — only evaluated when referenced
  # let! is eager — evaluated before each example

  describe "#add" do
    it { expect(calculator.add(a, b)).to eq(5) }
  end

  describe "#divide" do
    context "when dividing by zero" do
      let(:b) { 0 }

      it "raises an error" do
        expect { calculator.divide(a, b) }.to raise_error(ZeroDivisionError)
      end
    end
  end
end
```

## Testing HTTP with WebMock and VCR

### WebMock

```ruby
require "webmock/rspec"

RSpec.describe ApiClient do
  before do
    stub_request(:get, "https://api.example.com/users")
      .with(headers: { "Authorization" => "Bearer token123" })
      .to_return(
        status: 200,
        body: [{ id: 1, name: "Alice" }].to_json,
        headers: { "Content-Type" => "application/json" }
      )
  end

  it "fetches users" do
    client = described_class.new(token: "token123")
    users = client.fetch_users
    expect(users.first["name"]).to eq("Alice")
  end
end
```

### VCR

```ruby
# spec/support/vcr.rb
VCR.configure do |config|
  config.cassette_library_dir = "spec/cassettes"
  config.hook_into :webmock
  config.filter_sensitive_data("<API_KEY>") { ENV.fetch("API_KEY") }
end

RSpec.describe ApiClient do
  it "fetches users", vcr: { cassette_name: "api/users" } do
    users = described_class.new.fetch_users
    expect(users).not_to be_empty
  end
end
```

## Test Organization

### Directory Structure

```
spec/
├── spec_helper.rb
├── rails_helper.rb          # Rails projects only
├── support/
│   ├── factory_bot.rb
│   ├── shared_examples/
│   └── shared_contexts/
├── factories/
│   ├── users.rb
│   └── orders.rb
├── models/
│   └── user_spec.rb
├── services/
│   └── create_user_spec.rb
├── requests/                # API/integration tests
│   └── users_spec.rb
└── system/                  # E2E tests (Rails)
    └── user_registration_spec.rb
```

## Running Tests

```bash
# Run all tests
bundle exec rspec

# Run specific file
bundle exec rspec spec/models/user_spec.rb

# Run specific test by line number
bundle exec rspec spec/models/user_spec.rb:42

# Run by tag
bundle exec rspec --tag focus
bundle exec rspec --tag ~slow

# Run with documentation format
bundle exec rspec --format documentation

# Run with coverage
COVERAGE=true bundle exec rspec

# Profile slow tests
bundle exec rspec --profile 10

# Run failed tests only
bundle exec rspec --only-failures

# Parallel tests
bundle exec parallel_rspec spec/
```

## Best Practices

### DO

- **Follow TDD**: Write tests before code (red-green-refactor)
- **Test one thing**: Each `it` block verifies a single behavior
- **Use descriptive names**: Context reads like a sentence
- **Use factories**: Prefer `factory_bot` over fixtures
- **Mock external dependencies**: Don't depend on external services
- **Test edge cases**: nil, empty, boundary conditions
- **Aim for 80%+ coverage**: Focus on critical paths

### DON'T

- **Don't test implementation**: Test behavior, not internals
- **Don't use `before(:all)`**: Leads to shared state between tests
- **Don't stub the system under test**: Only stub collaborators
- **Don't use `allow_any_instance_of`**: Use dependency injection instead
- **Don't write brittle tests**: Avoid over-specific mocks
- **Don't ignore test failures**: All tests must pass

## Quick Reference

| Pattern | Usage |
|---------|-------|
| `describe` | Group by class or method |
| `context` | Group by scenario |
| `it` | Single expectation |
| `let` | Lazy-evaluated variable |
| `subject` | Object under test |
| `before/after` | Setup/teardown hooks |
| `build/create` | Factory methods |
| `allow/expect` | Stub/mock methods |
| `shared_examples` | Reusable test groups |
| `instance_double` | Verified test double |

**Remember**: Good tests are documentation. They describe what the code does, not how it does it.
