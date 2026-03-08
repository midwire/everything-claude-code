---
name: rails-tdd
description: Rails testing with RSpec, request specs, system specs with Capybara, factory_bot, shoulda-matchers, and TDD methodology.
origin: ECC
---

# Rails Testing with TDD

Test-driven development for Ruby on Rails applications using RSpec, Capybara, and factory_bot.

## When to Activate

- Writing new Rails features
- Implementing Rails controllers and APIs
- Testing ActiveRecord models
- Setting up testing infrastructure for Rails projects

## Setup

### Gemfile

```ruby
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end

group :test do
  gem "shoulda-matchers"
  gem "capybara"
  gem "selenium-webdriver"
  gem "simplecov", require: false
  gem "webmock"
  gem "vcr"
  gem "database_cleaner-active_record"
end
```

### spec/rails_helper.rb

```ruby
require "simplecov"
SimpleCov.start "rails" do
  minimum_coverage 80
end

require "spec_helper"

ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"

abort("Running in production!") if Rails.env.production?
require "rspec/rails"

Dir[Rails.root.join("spec/support/**/*.rb")].each { |f| require f }

RSpec.configure do |config|
  config.fixture_paths = [Rails.root.join("spec/fixtures")]
  config.use_transactional_fixtures = true
  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!

  config.include FactoryBot::Syntax::Methods
end

Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

## Model Specs

```ruby
# spec/models/user_spec.rb
RSpec.describe User do
  describe "validations" do
    it { is_expected.to validate_presence_of(:email) }
    it { is_expected.to validate_uniqueness_of(:email).case_insensitive }
    it { is_expected.to validate_length_of(:name).is_at_most(100) }
  end

  describe "associations" do
    it { is_expected.to have_many(:orders).dependent(:destroy) }
    it { is_expected.to belong_to(:organization) }
    it { is_expected.to have_one(:profile).dependent(:destroy) }
  end

  describe "scopes" do
    describe ".active" do
      it "returns only active users" do
        active_user = create(:user, active: true)
        create(:user, active: false)

        expect(described_class.active).to contain_exactly(active_user)
      end
    end
  end

  describe "#full_name" do
    it "returns first and last name" do
      user = build(:user, first_name: "Alice", last_name: "Smith")
      expect(user.full_name).to eq("Alice Smith")
    end

    it "handles missing last name" do
      user = build(:user, first_name: "Alice", last_name: nil)
      expect(user.full_name).to eq("Alice")
    end
  end
end
```

## Request Specs (API Testing)

```ruby
# spec/requests/api/v1/orders_spec.rb
RSpec.describe "Api::V1::Orders" do
  let(:user) { create(:user) }
  let(:headers) { auth_headers_for(user) }

  describe "GET /api/v1/orders" do
    it "returns user orders" do
      orders = create_list(:order, 3, user: user)
      create(:order) # Another user's order

      get "/api/v1/orders", headers: headers

      expect(response).to have_http_status(:ok)
      expect(response.parsed_body.size).to eq(3)
    end

    it "returns 401 without authentication" do
      get "/api/v1/orders"

      expect(response).to have_http_status(:unauthorized)
    end
  end

  describe "POST /api/v1/orders" do
    let(:product) { create(:product, price: 29.99) }
    let(:valid_params) { { order: { product_id: product.id, quantity: 2 } } }

    it "creates an order" do
      expect {
        post "/api/v1/orders", params: valid_params, headers: headers
      }.to change(Order, :count).by(1)

      expect(response).to have_http_status(:created)
      expect(response.parsed_body["total"]).to eq("59.98")
    end

    it "returns errors for invalid params" do
      post "/api/v1/orders", params: { order: { quantity: -1 } }, headers: headers

      expect(response).to have_http_status(:unprocessable_entity)
      expect(response.parsed_body["errors"]).to be_present
    end
  end
end
```

## System Specs (E2E with Capybara)

```ruby
# spec/system/user_registration_spec.rb
RSpec.describe "User Registration", type: :system do
  before { driven_by(:selenium_chrome_headless) }

  it "allows a new user to register" do
    visit new_user_registration_path

    fill_in "Email", with: "alice@example.com"
    fill_in "Password", with: "SecurePassword123!"
    fill_in "Password confirmation", with: "SecurePassword123!"
    click_button "Sign up"

    expect(page).to have_text("Welcome! You have signed up successfully.")
    expect(User.find_by(email: "alice@example.com")).to be_present
  end

  it "shows errors for invalid registration" do
    visit new_user_registration_path

    fill_in "Email", with: "invalid"
    click_button "Sign up"

    expect(page).to have_text("Email is invalid")
  end
end
```

## Service Specs

```ruby
# spec/services/orders/create_spec.rb
RSpec.describe Orders::Create do
  subject(:service) { described_class.new(user: user, cart: cart, payment_gateway: gateway) }

  let(:user) { create(:user) }
  let(:cart) { create(:cart, :with_items, user: user) }
  let(:gateway) { instance_double(Stripe::Gateway) }

  describe "#call" do
    context "with successful payment" do
      before do
        allow(gateway).to receive(:charge).and_return(
          OpenStruct.new(success?: true, payment_id: "pi_123")
        )
      end

      it "creates an order" do
        expect { service.call }.to change(Order, :count).by(1)
      end

      it "clears the cart" do
        service.call
        expect(cart.items.reload).to be_empty
      end

      it "sends confirmation email" do
        expect { service.call }
          .to have_enqueued_mail(OrderMailer, :confirmation)
      end
    end

    context "with failed payment" do
      before do
        allow(gateway).to receive(:charge).and_return(
          OpenStruct.new(success?: false, error: "Card declined")
        )
      end

      it "raises PaymentError" do
        expect { service.call }.to raise_error(PaymentError, "Card declined")
      end

      it "does not create an order" do
        expect {
          service.call rescue PaymentError
        }.not_to change(Order, :count)
      end
    end
  end
end
```

## Job Specs

```ruby
# spec/jobs/process_order_job_spec.rb
RSpec.describe ProcessOrderJob do
  let(:order) { create(:order) }

  it "processes the order" do
    service = instance_double(Orders::Process, call: true)
    allow(Orders::Process).to receive(:new).with(order: order).and_return(service)

    described_class.perform_now(order.id)

    expect(service).to have_received(:call)
  end

  it "skips already processed orders" do
    order.update!(processed_at: Time.current)

    expect(Orders::Process).not_to receive(:new)

    described_class.perform_now(order.id)
  end

  it "retries on deadlock" do
    expect(described_class).to have_attributes(
      retry_on: include(ActiveRecord::Deadlocked)
    )
  end
end
```

## Mailer Specs

```ruby
# spec/mailers/order_mailer_spec.rb
RSpec.describe OrderMailer do
  describe "#confirmation" do
    let(:order) { create(:order) }
    let(:mail) { described_class.confirmation(order) }

    it "sends to the user's email" do
      expect(mail.to).to eq([order.user.email])
    end

    it "includes the order number" do
      expect(mail.body.encoded).to include(order.number)
    end

    it "has the correct subject" do
      expect(mail.subject).to eq("Order Confirmation ##{order.number}")
    end
  end
end
```

## Test Helpers

```ruby
# spec/support/auth_helpers.rb
module AuthHelpers
  def auth_headers_for(user)
    token = user.generate_jwt_token
    { "Authorization" => "Bearer #{token}" }
  end

  def sign_in_as(user)
    post login_path, params: { email: user.email, password: "password" }
  end
end

RSpec.configure do |config|
  config.include AuthHelpers
end
```

## Running Tests

```bash
# Run all specs
bundle exec rspec

# Run by type
bundle exec rspec spec/models/
bundle exec rspec spec/requests/
bundle exec rspec spec/system/

# Run specific spec
bundle exec rspec spec/models/user_spec.rb:42

# With documentation format
bundle exec rspec --format documentation

# With coverage
COVERAGE=true bundle exec rspec

# Parallel
bundle exec parallel_rspec spec/
```

## Quick Reference

| Spec Type | Location | Tests |
|-----------|----------|-------|
| Model | `spec/models/` | Validations, scopes, methods |
| Request | `spec/requests/` | API endpoints, HTTP responses |
| System | `spec/system/` | Full browser E2E flows |
| Service | `spec/services/` | Business logic |
| Job | `spec/jobs/` | Background job behavior |
| Mailer | `spec/mailers/` | Email content and delivery |
| Policy | `spec/policies/` | Authorization rules |

**Remember**: Write the test first. Watch it fail. Write the minimum code to make it pass. Refactor. Repeat.
