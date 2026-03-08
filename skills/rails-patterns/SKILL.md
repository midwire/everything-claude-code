---
name: rails-patterns
description: Ruby on Rails architecture patterns, ActiveRecord best practices, controllers, service objects, concerns, Hotwire/Turbo, and production-grade Rails apps.
origin: ECC
---

# Rails Development Patterns

Production-grade Rails architecture patterns for scalable, maintainable applications.

## When to Activate

- Building Ruby on Rails applications
- Working with ActiveRecord models and queries
- Designing Rails controllers and APIs
- Implementing background jobs
- Using Hotwire/Turbo for modern Rails frontend

## Project Structure

### Recommended Layout

```
app/
├── controllers/
│   ├── application_controller.rb
│   ├── concerns/
│   └── api/
│       └── v1/
├── models/
│   ├── application_record.rb
│   └── concerns/
├── services/              # Service objects
├── queries/               # Query objects
├── forms/                 # Form objects
├── presenters/            # View presenters
├── jobs/                  # Background jobs
├── mailers/
├── serializers/           # API serializers
├── policies/              # Authorization (Pundit)
└── views/
    ├── layouts/
    └── components/        # ViewComponent
```

## ActiveRecord Best Practices

### Model Organization

```ruby
# frozen_string_literal: true

class User < ApplicationRecord
  # 1. Constants
  VALID_ROLES = %w[admin member viewer].freeze

  # 2. Associations
  has_many :orders, dependent: :destroy
  has_many :products, through: :orders
  has_one :profile, dependent: :destroy
  belongs_to :organization

  # 3. Validations
  validates :email, presence: true, uniqueness: { case_sensitive: false }
  validates :name, presence: true, length: { maximum: 100 }
  validates :role, inclusion: { in: VALID_ROLES }

  # 4. Scopes
  scope :active, -> { where(active: true) }
  scope :admins, -> { where(role: "admin") }
  scope :recent, -> { order(created_at: :desc) }
  scope :search, ->(query) { where("name ILIKE ?", "%#{sanitize_sql_like(query)}%") }

  # 5. Callbacks (use sparingly)
  before_validation :normalize_email

  # 6. Enums
  enum :status, { pending: 0, active: 1, suspended: 2 }, prefix: true

  # 7. Class methods
  def self.find_by_credentials(email, password)
    user = find_by(email: email.downcase)
    user&.authenticate(password)
  end

  # 8. Instance methods
  def admin?
    role == "admin"
  end

  def full_name
    "#{first_name} #{last_name}".strip
  end

  private

  def normalize_email
    self.email = email&.downcase&.strip
  end
end
```

### Query Optimization

```ruby
# Bad: N+1 queries
User.all.each { |u| puts u.orders.count }

# Good: Eager loading
User.includes(:orders).each { |u| puts u.orders.size }

# select_related equivalent (single query with JOIN)
User.joins(:profile).select("users.*, profiles.bio")

# Preload for separate queries (better for large datasets)
User.preload(:orders, :profile)

# Eager load (LEFT OUTER JOIN — loads association in same query)
User.eager_load(:orders).where(orders: { status: :active })

# Counter cache
# migration: add_column :users, :orders_count, :integer, default: 0
belongs_to :user, counter_cache: true

# Batch processing
User.find_each(batch_size: 1000) do |user|
  user.process_data
end
```

### Scopes and Querying

```ruby
class Order < ApplicationRecord
  scope :placed_between, ->(start_date, end_date) {
    where(created_at: start_date..end_date)
  }

  scope :with_total_above, ->(amount) {
    where("total_amount > ?", amount)
  }

  scope :by_status, ->(status) {
    where(status: status) if status.present?
  }

  # Chainable
  # Order.placed_between(1.week.ago, Time.current).with_total_above(100)
end
```

## Service Objects

```ruby
# frozen_string_literal: true

class Orders::Create
  def initialize(user:, cart:, payment_gateway: Stripe::Gateway.new)
    @user = user
    @cart = cart
    @payment_gateway = payment_gateway
  end

  def call
    ActiveRecord::Base.transaction do
      order = create_order
      charge_payment(order)
      send_confirmation(order)
      clear_cart
      order
    end
  end

  private

  attr_reader :user, :cart, :payment_gateway

  def create_order
    Order.create!(
      user: user,
      items: cart.items.map { |item| build_order_item(item) },
      total: cart.total
    )
  end

  def charge_payment(order)
    result = payment_gateway.charge(amount: order.total, customer: user.stripe_id)
    raise PaymentError, result.error unless result.success?

    order.update!(payment_id: result.payment_id)
  end

  def send_confirmation(order)
    OrderMailer.confirmation(order).deliver_later
  end

  def clear_cart
    cart.items.destroy_all
  end

  def build_order_item(cart_item)
    OrderItem.new(
      product: cart_item.product,
      quantity: cart_item.quantity,
      price: cart_item.product.price
    )
  end
end
```

## Controller Patterns

```ruby
# frozen_string_literal: true

class Api::V1::OrdersController < ApplicationController
  before_action :authenticate_user!
  before_action :set_order, only: %i[show update destroy]

  def index
    orders = current_user.orders
                         .includes(:items, :products)
                         .order(created_at: :desc)
                         .page(params[:page])

    render json: orders, each_serializer: OrderSerializer
  end

  def create
    order = Orders::Create.new(user: current_user, cart: current_cart).call

    render json: order, serializer: OrderSerializer, status: :created
  rescue Orders::Create::PaymentError => e
    render json: { error: e.message }, status: :unprocessable_entity
  end

  def show
    render json: @order, serializer: OrderDetailSerializer
  end

  private

  def set_order
    @order = current_user.orders.find(params[:id])
  end
end
```

## Concerns

```ruby
# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  included do
    scope :search, ->(query) {
      return all if query.blank?

      where(
        searchable_columns.map { |col| "#{col} ILIKE :q" }.join(" OR "),
        q: "%#{sanitize_sql_like(query)}%"
      )
    }
  end

  class_methods do
    def searchable_columns
      @searchable_columns || %w[name]
    end

    def searchable(*columns)
      @searchable_columns = columns.map(&:to_s)
    end
  end
end

# Usage
class Product < ApplicationRecord
  include Searchable
  searchable :name, :description, :sku
end

Product.search("widget")
```

## Background Jobs

```ruby
# app/jobs/process_order_job.rb
class ProcessOrderJob < ApplicationJob
  queue_as :default
  retry_on ActiveRecord::Deadlocked, wait: 5.seconds, attempts: 3
  discard_on ActiveJob::DeserializationError

  def perform(order_id)
    order = Order.find(order_id)
    return if order.processed?

    Orders::Process.new(order: order).call
  end
end

# Enqueue
ProcessOrderJob.perform_later(order.id)
ProcessOrderJob.set(wait: 5.minutes).perform_later(order.id)
```

## Hotwire / Turbo

```ruby
# Controller with Turbo Stream responses
class CommentsController < ApplicationController
  def create
    @comment = @post.comments.build(comment_params)

    if @comment.save
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to @post }
      end
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

```erb
<%# app/views/comments/create.turbo_stream.erb %>
<%= turbo_stream.append "comments" do %>
  <%= render @comment %>
<% end %>

<%= turbo_stream.update "comment_form" do %>
  <%= render "form", comment: Comment.new %>
<% end %>
```

## Configuration

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.time_zone = "UTC"
    config.active_record.default_timezone = :utc

    # API-only mode
    config.api_only = true

    # Autoload paths
    config.autoload_paths += %W[#{config.root}/app/services]
    config.autoload_paths += %W[#{config.root}/app/queries]
  end
end
```

## Quick Reference

| Pattern | When to Use |
|---------|-------------|
| Service object | Multi-step business logic |
| Query object | Complex database queries |
| Form object | Multi-model forms, virtual attributes |
| Presenter | Complex view logic |
| Policy | Authorization rules (Pundit) |
| Concern | Shared model/controller behavior |
| Job | Async/background processing |
| Serializer | API response formatting |

**Remember**: Rails conventions are powerful — follow them unless you have a compelling reason not to. Fat models are better than fat controllers, but service objects are better than either.
