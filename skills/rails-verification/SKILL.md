---
name: rails-verification
description: "Verification loop for Rails projects: migrations, RuboCop, RSpec with coverage, Brakeman security scans, and deployment readiness checks."
origin: ECC
---

# Rails Verification Loop

Run before PRs, after major changes, and pre-deploy to ensure Rails application quality and security.

## When to Activate

- Before opening a pull request for a Rails project
- After major model changes, migration updates, or dependency upgrades
- Pre-deployment verification for staging or production
- Running full lint -> test -> security -> deploy readiness pipeline

## Phase 1: Environment Check

```bash
# Verify Ruby version
ruby --version                          # Should match .ruby-version
cat .ruby-version 2>/dev/null

# Check bundle
bundle check || bundle install

# Verify database
bin/rails db:version
```

## Phase 2: Migration Safety

```bash
# Check for pending migrations
bin/rails db:migrate:status

# Run migrations
bin/rails db:migrate

# Verify schema is committed
git diff --name-only db/schema.rb db/structure.sql

# Rollback test (verify reversibility)
bin/rails db:rollback STEP=1
bin/rails db:migrate
```

### Migration Checklist

- [ ] Migrations are reversible (or have explicit `down` method)
- [ ] Large table changes use `disable_ddl_transaction!` if needed
- [ ] Indexes added concurrently for large tables: `add_index :table, :col, algorithm: :concurrently`
- [ ] No data migrations mixed with schema migrations
- [ ] Column removals have `ignored_columns` set first

## Phase 3: Linting

```bash
# RuboCop
rubocop --format simple
rubocop --auto-correct-all              # Fix auto-correctable offenses

# Check for common issues
grep -rn "binding.pry\|binding.irb\|byebug\|debugger" app/ lib/ --include="*.rb"
grep -rn "puts \|p \|pp " app/ lib/ --include="*.rb" | grep -v "# " | head -20
```

## Phase 4: Test Suite

```bash
# Run full test suite with coverage
COVERAGE=true bundle exec rspec

# Run with documentation format for visibility
bundle exec rspec --format documentation

# Check coverage report
open coverage/index.html 2>/dev/null || echo "Check coverage/index.html"

# Run parallel tests (if configured)
bundle exec parallel_rspec spec/
```

### Test Checklist

- [ ] All tests pass
- [ ] Coverage >= 80%
- [ ] No skipped/pending tests without justification
- [ ] No slow tests without `@slow` tag
- [ ] Request/system specs for new endpoints

## Phase 5: Security Scan

```bash
# Brakeman static analysis
brakeman --no-pager --confidence-level 2

# Dependency vulnerabilities
bundle audit check --update

# Check for known CVEs
bundle audit

# Verify no secrets in codebase
git log --diff-filter=A --name-only -- '*.env' '*.key' '*.pem' | head -20
grep -rn "password\s*=" config/ --include="*.rb" | grep -v "password_" | head -10
```

### Security Checklist

- [ ] Brakeman reports no high-confidence issues
- [ ] `bundle audit` is clean
- [ ] No hardcoded secrets in source
- [ ] Strong parameters used in all controllers
- [ ] CSRF protection enabled

## Phase 6: Asset & Build Check

```bash
# Precompile assets (if applicable)
bin/rails assets:precompile RAILS_ENV=production 2>&1 | tail -5

# Check for JavaScript/CSS issues
bin/rails assets:clean
```

## Phase 7: Deployment Readiness

```bash
# Production config check
RAILS_ENV=production bin/rails runner "puts 'Config OK'" 2>&1

# Verify required env vars
ruby -e "
  required = %w[DATABASE_URL SECRET_KEY_BASE RAILS_ENV]
  missing = required.select { |v| ENV[v].nil? || ENV[v].empty? }
  if missing.any?
    puts 'Missing: #{missing.join(', ')}'
  else
    puts 'All required env vars present'
  end
"
```

## Quick Verification (Pre-Commit)

```bash
# Fast check — run before every commit
rubocop --only Layout,Style --format simple && \
bundle exec rspec --fail-fast --format progress && \
echo "Pre-commit checks passed"
```

## Full Verification (Pre-PR)

```bash
# Complete pipeline
rubocop --format simple && \
bundle exec rspec --format documentation && \
brakeman --no-pager --confidence-level 2 && \
bundle audit check --update && \
echo "All checks passed — ready for PR"
```

## Verification Results

Report format:

```text
Phase 1: Environment    ✓ Ruby 3.3.0, Rails 7.1.3
Phase 2: Migrations     ✓ No pending, schema committed
Phase 3: Linting        ✓ RuboCop clean (0 offenses)
Phase 4: Tests          ✓ 142 examples, 0 failures (87% coverage)
Phase 5: Security       ✓ Brakeman clean, bundle audit clean
Phase 6: Assets         ✓ Precompilation successful
Phase 7: Deploy Ready   ✓ All env vars present

Overall: PASS — Ready for PR/deploy
```
