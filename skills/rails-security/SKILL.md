---
name: rails-security
description: Rails security best practices, strong parameters, authentication, authorization, CSRF protection, SQL injection prevention, and secure deployment.
origin: ECC
---

# Rails Security Best Practices

Comprehensive security guidelines for Ruby on Rails applications.

## When to Activate

- Setting up Rails authentication and authorization
- Implementing user permissions and roles
- Configuring production security settings
- Reviewing Rails application for security issues
- Deploying Rails applications to production

## Strong Parameters

```ruby
# Always use strong params — never pass raw params to models
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:name, :email, :role,
                                  address: [:street, :city, :state, :zip])
  end
end

# Never do this:
User.create(params[:user])                    # Mass assignment vulnerability
User.create(params[:user].permit!)            # Permits everything
```

## Authentication

### has_secure_password

```ruby
# Gemfile
gem "bcrypt"

# Model
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: { case_sensitive: false }
  validates :password, length: { minimum: 12 }, if: :password_digest_changed?
end

# Controller
class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email]&.downcase)

    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to root_path
    else
      # Generic message — don't reveal which field is wrong
      flash.now[:alert] = "Invalid email or password"
      render :new, status: :unprocessable_entity
    end
  end

  def destroy
    reset_session
    redirect_to root_path
  end
end
```

### Devise Best Practices

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  config.stretches = 12                    # bcrypt cost factor
  config.password_length = 12..128
  config.lock_strategy = :failed_attempts
  config.maximum_attempts = 5
  config.unlock_strategy = :time
  config.unlock_in = 30.minutes
  config.timeout_in = 30.minutes
  config.remember_for = 2.weeks
end
```

## Authorization

### Pundit

```ruby
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def update?
    record.user == user || user.admin?
  end

  def destroy?
    user.admin?
  end

  class Scope < Scope
    def resolve
      if user.admin?
        scope.all
      else
        scope.where(user: user)
      end
    end
  end
end

# Controller
class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    authorize @post

    if @post.update(post_params)
      redirect_to @post
    else
      render :edit
    end
  end

  def index
    @posts = policy_scope(Post)
  end
end
```

## SQL Injection Prevention

```ruby
# SAFE: ActiveRecord parameterizes automatically
User.where(email: params[:email])
User.where("email = ?", params[:email])
User.where("email = :email", email: params[:email])

# VULNERABLE: String interpolation
User.where("email = '#{params[:email]}'")              # SQL injection!
User.where("email = " + params[:email])                # SQL injection!
User.order(params[:sort])                              # SQL injection!

# Safe ordering
ALLOWED_SORT = %w[name created_at email].freeze

def safe_order(column)
  ALLOWED_SORT.include?(column) ? column : "created_at"
end

User.order(safe_order(params[:sort]))
```

## XSS Prevention

```erb
<%# SAFE: Rails auto-escapes by default %>
<%= user.name %>

<%# DANGEROUS: raw/html_safe bypasses escaping %>
<%= raw user.bio %>              <%# XSS vulnerability! %>
<%= user.bio.html_safe %>        <%# XSS vulnerability! %>

<%# SAFE: Sanitize user HTML %>
<%= sanitize user.bio, tags: %w[b i em strong p br] %>

<%# SAFE: Content tag helpers %>
<%= content_tag :span, user.name, class: "username" %>
```

## CSRF Protection

```ruby
# ApplicationController — enabled by default
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end

# API controllers — use null session or token auth
class Api::ApplicationController < ActionController::API
  # No CSRF needed for stateless API with token auth
end

# Skip CSRF for webhooks (verify signature instead)
class WebhooksController < ApplicationController
  skip_forgery_protection

  before_action :verify_webhook_signature

  private

  def verify_webhook_signature
    signature = request.headers["X-Webhook-Signature"]
    payload = request.body.read
    expected = OpenSSL::HMAC.hexdigest("SHA256", webhook_secret, payload)

    head :unauthorized unless ActiveSupport::SecurityUtils.secure_compare(signature, expected)
  end
end
```

## Secret Management

```ruby
# Use Rails credentials (encrypted)
# Edit: rails credentials:edit
Rails.application.credentials.secret_key_base
Rails.application.credentials.dig(:aws, :access_key_id)

# Or use environment variables
ENV.fetch("DATABASE_URL")           # Raises if missing
ENV.fetch("PORT", "3000")           # Default for non-sensitive values

# Never hardcode secrets
config.api_key = "sk_live_abc123"   # NEVER DO THIS
```

## Security Headers

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.script_src  :self
    policy.style_src   :self, :unsafe_inline
    policy.img_src     :self, :data, :https
    policy.connect_src :self
    policy.font_src    :self
    policy.frame_src   :none
  end

  config.content_security_policy_nonce_generator = ->(request) {
    request.session.id.to_s
  }
end

# config/environments/production.rb
config.force_ssl = true
config.ssl_options = { hsts: { subdomains: true, preload: true, expires: 1.year } }
```

## Rate Limiting (Rails 8+)

```ruby
class SessionsController < ApplicationController
  rate_limit to: 10, within: 3.minutes, only: :create,
             with: -> { redirect_to new_session_url, alert: "Try again later." }
end

# Or with Rack::Attack
class Rack::Attack
  throttle("logins/ip", limit: 5, period: 60.seconds) do |req|
    req.ip if req.path == "/login" && req.post?
  end

  throttle("logins/email", limit: 5, period: 60.seconds) do |req|
    req.params["email"]&.downcase if req.path == "/login" && req.post?
  end
end
```

## Security Scanning

```bash
# Brakeman — static analysis for Rails
brakeman --no-pager
brakeman --confidence-level 2    # Only high-confidence issues
brakeman -o brakeman-report.html # HTML report

# bundler-audit — dependency vulnerabilities
bundle audit check --update

# Check for insecure gems
bundle audit
```

## Secure File Uploads

```ruby
# Use Active Storage with validation
class User < ApplicationRecord
  has_one_attached :avatar

  validate :acceptable_avatar

  private

  def acceptable_avatar
    return unless avatar.attached?

    unless avatar.content_type.in?(%w[image/jpeg image/png image/webp])
      errors.add(:avatar, "must be a JPEG, PNG, or WebP")
    end

    if avatar.byte_size > 5.megabytes
      errors.add(:avatar, "must be less than 5MB")
    end
  end
end
```

## Production Checklist

| Check | Description |
|-------|-------------|
| `config.force_ssl = true` | Force HTTPS in production |
| `protect_from_forgery` | CSRF protection enabled |
| Strong parameters | All controllers use `permit` |
| `SECRET_KEY_BASE` | Set via env var, not checked in |
| Brakeman clean | No high-confidence warnings |
| `bundle audit` clean | No known vulnerabilities |
| Rate limiting | Login, signup, API endpoints throttled |
| CSP headers | Content Security Policy configured |
| Logging | No sensitive data in logs |
| Password policy | Minimum 12 characters, bcrypt cost >= 12 |

**Remember**: Rails provides excellent security defaults — don't disable them without understanding the consequences.
