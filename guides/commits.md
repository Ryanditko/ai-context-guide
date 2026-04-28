# Commit Conventions

Guide for writing clear, semantic, and useful commit messages following the Conventional Commits standard.

## Why Good Commits Matter

- **Searchable history**: Find when and why changes were made
- **Automated changelogs**: Generate release notes automatically
- **Clear communication**: Team understands changes without reading code
- **Easier debugging**: `git bisect` works better with atomic commits
- **Better reviews**: Reviewers understand intent before reading code

## Conventional Commits Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Components

| Component | Required | Description |
|-----------|----------|-------------|
| `type` | Yes | Category of change |
| `scope` | No | Module/component affected |
| `description` | Yes | Short summary (imperative mood) |
| `body` | No | Detailed explanation |
| `footer` | No | Breaking changes, issue refs |

## Commit Types

| Type | Description | Changelog Section |
|------|-------------|-------------------|
| `feat` | New feature for users | Features |
| `fix` | Bug fix for users | Bug Fixes |
| `docs` | Documentation only | Documentation |
| `style` | Formatting, no code change | - |
| `refactor` | Code change, no feature/fix | - |
| `perf` | Performance improvement | Performance |
| `test` | Adding/fixing tests | - |
| `build` | Build system, dependencies | - |
| `ci` | CI configuration | - |
| `chore` | Maintenance tasks | - |
| `revert` | Revert previous commit | Reverts |

## Rules

### 1. Description Rules

```bash
# ✅ Good
feat: add user authentication endpoint
fix: resolve null pointer in payment processing

# ❌ Bad
feat: Add User Authentication Endpoint  # Capitalized
fix: resolved null pointer              # Past tense
feat: add user authentication endpoint. # Period at end
```

| Rule | Correct | Incorrect |
|------|---------|-----------|
| Start lowercase | `add feature` | `Add feature` |
| No period at end | `add feature` | `add feature.` |
| Imperative mood | `add`, `fix`, `change` | `added`, `fixes`, `changed` |
| Max 72 characters | Short and clear | Long rambling description that goes on and on |

### 2. Scope Guidelines

```bash
# By module/feature
feat(auth): implement JWT refresh tokens
fix(payments): handle timeout in Stripe calls
test(orders): add integration tests for checkout

# By layer
fix(api): validate request body schema
refactor(db): optimize user queries
feat(ui): add loading spinner component

# By file type
docs(readme): update installation instructions
style(css): fix button alignment
```

### 3. Body Guidelines

Use the body to explain **what** and **why**, not **how**.

```bash
fix(auth): resolve session timeout during checkout

The session was expiring mid-checkout because the activity
tracker wasn't being updated during payment processing.

This fix ensures the session is refreshed when the payment
form is submitted, preventing users from losing their cart.

Closes #456
```

### 4. Footer Guidelines

```bash
# Reference issues
Closes #123
Fixes #456
Refs #789

# Breaking changes
BREAKING CHANGE: The `user` endpoint now returns an object instead of array

# Multiple footers
Closes #123
Reviewed-by: John Doe
Co-authored-by: Jane Smith <jane@example.com>
```

## Examples by Scenario

### Simple Feature

```bash
feat: add password strength indicator
```

### Feature with Scope

```bash
feat(auth): implement OAuth2 login with Google

Add Google OAuth2 provider to the authentication system.
Users can now sign in with their Google accounts.

- Added Google OAuth2 configuration
- Created callback handler
- Updated login page with Google button

Closes #234
```

### Bug Fix

```bash
fix(cart): prevent duplicate items when clicking fast

Race condition allowed duplicate items when add-to-cart
was clicked multiple times quickly. Added debounce and
optimistic locking to prevent duplicates.

Fixes #567
```

### Breaking Change

```bash
feat(api)!: change user endpoint response format

BREAKING CHANGE: GET /users now returns paginated response

Before:
  [{ "id": 1, "name": "John" }]

After:
  {
    "data": [{ "id": 1, "name": "John" }],
    "pagination": { "page": 1, "total": 100 }
  }

Migration: Update all clients to handle the new response structure.
```

### Refactoring

```bash
refactor(orders): extract pricing logic to dedicated service

Moved pricing calculations from OrderService to PricingService
to improve separation of concerns and testability.

No functional changes - all existing tests pass.
```

### Performance Improvement

```bash
perf(db): add index for order status queries

Added composite index on (user_id, status, created_at) to
optimize the order history page query.

Before: 850ms average
After: 45ms average

Closes #890
```

### Documentation

```bash
docs(api): add authentication section to API docs

- Added JWT token flow explanation
- Included request/response examples
- Documented error codes
```

### Revert

```bash
revert: feat(payments): add Apple Pay support

This reverts commit abc123def456.

Apple Pay integration is causing checkout failures on
Safari 16. Reverting until the issue is investigated.

Refs #901
```

## Anti-patterns and Fixes

### Vague Messages

```bash
# ❌ Bad
fix: fix bug
update: changes
feat: stuff

# ✅ Good
fix(auth): resolve login failure for users with special characters in password
update(deps): upgrade React from 17 to 18
feat(search): add fuzzy matching for product names
```

### Too Many Changes

```bash
# ❌ Bad (multiple unrelated changes)
feat: add user auth, fix payment bug, update readme

# ✅ Good (separate commits)
feat(auth): add user authentication
fix(payments): resolve timeout handling
docs(readme): update installation steps
```

### WIP Commits

```bash
# ❌ Bad
WIP
WIP 2
fix stuff
asdfasdf

# ✅ Good (squash before PR)
git rebase -i HEAD~4
# Combine into single meaningful commit
feat(checkout): implement guest checkout flow
```

### Describing How Instead of Why

```bash
# ❌ Bad (describes implementation)
fix: add if statement to check for null

# ✅ Good (describes intent)
fix(orders): handle missing shipping address gracefully

Users without a saved address were getting a 500 error.
Now shows a friendly prompt to add an address.
```

## Commit Workflow

### Before Committing

```bash
# 1. Review your changes
git diff --staged

# 2. Ensure tests pass
npm test

# 3. Check for lint errors
npm run lint

# 4. Write meaningful commit message
git commit
```

### Interactive Commit (with Commitizen)

```bash
# Install
npm install -g commitizen

# Use
git cz

# Prompts:
# ? Select type: feat
# ? Scope: auth
# ? Short description: add password reset flow
# ? Longer description: (optional)
# ? Breaking changes: (optional)
# ? Issues closed: #123
```

### Amending Commits

```bash
# Fix last commit message
git commit --amend -m "feat(auth): correct message here"

# Add forgotten files to last commit
git add forgotten-file.js
git commit --amend --no-edit

# ⚠️ Never amend pushed commits on shared branches
```

### Squashing Commits

```bash
# Before PR: Clean up messy commit history
git rebase -i HEAD~5

# In editor, change 'pick' to 'squash' for commits to combine
pick abc123 feat(auth): implement login
squash def456 WIP
squash ghi789 fix typo
squash jkl012 more fixes

# Result: Single clean commit
```

## Automation Tools

### Commitlint

```bash
# Install
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Configure (commitlint.config.js)
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'scope-enum': [2, 'always', ['auth', 'api', 'ui', 'db', 'core']],
    'subject-case': [2, 'always', 'lower-case'],
  },
};
```

### Husky (Git Hooks)

```bash
# Install
npm install --save-dev husky

# Setup
npx husky install

# Add commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "$1"'
```

### Pre-commit Config

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.12.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

### Automated Changelog

```bash
# Install standard-version
npm install --save-dev standard-version

# Generate changelog and bump version
npx standard-version

# Output: CHANGELOG.md
## [1.2.0] - 2024-01-15
### Features
- **auth:** add OAuth2 login with Google (#234)
### Bug Fixes
- **cart:** prevent duplicate items when clicking fast (#567)
```

## Commit Message Template

```bash
# Set up template
git config commit.template .gitmessage

# .gitmessage
# <type>(<scope>): <subject>
#
# <body>
#
# <footer>
#
# --- COMMIT TYPES ---
# feat:     New feature
# fix:      Bug fix
# docs:     Documentation
# style:    Formatting
# refactor: Code restructuring
# perf:     Performance
# test:     Tests
# build:    Build/deps
# ci:       CI config
# chore:    Maintenance
#
# --- RULES ---
# - Subject: lowercase, imperative, no period, max 72 chars
# - Body: explain what and why, not how
# - Footer: reference issues, note breaking changes
```

## Quick Reference

### Commit Checklist

- [ ] Type is correct (`feat`, `fix`, etc.)
- [ ] Scope is meaningful (if used)
- [ ] Description is lowercase, imperative, no period
- [ ] Description is under 72 characters
- [ ] Body explains why (if needed)
- [ ] Issues are referenced (if applicable)
- [ ] Breaking changes are noted with `!` and `BREAKING CHANGE:`

### Common Scopes

| Domain | Scopes |
|--------|--------|
| Features | `auth`, `payments`, `orders`, `users`, `search` |
| Layers | `api`, `ui`, `db`, `core`, `infra` |
| Config | `deps`, `ci`, `build`, `config` |
| Docs | `readme`, `api-docs`, `changelog` |
