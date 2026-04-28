# Git Workflow

Guide for Git workflow in teams.

## Branching Strategy

### Git Flow

```
main (production)
  │
  └── develop (integration)
        │
        ├── feature/user-auth
        ├── feature/payment-gateway
        │
        └── release/1.2.0
              │
              └── hotfix/critical-bug
```

### GitHub Flow (simpler)

```
main (always deployable)
  │
  ├── feature/add-login
  ├── fix/payment-bug
  └── chore/update-deps
```

### Trunk-Based Development

```
main (continuous deploy)
  │
  ├── short-lived-branch-1 (< 1 day)
  └── short-lived-branch-2 (< 1 day)
```

## Branch Naming Convention

### Pattern

```
<type>/<description>

# Common types
feature/    # New feature
fix/        # Bug fix
hotfix/     # Urgent production fix
chore/      # Maintenance, deps, configs
docs/       # Documentation
refactor/   # Refactoring
test/       # Adding tests
```

### Examples

```bash
# Good
feature/user-authentication
fix/payment-timeout-handling
hotfix/critical-security-patch
chore/upgrade-node-20
docs/api-documentation

# Bad
my-branch
fix
Feature/UserAuth  # Don't use uppercase
feature/add-new-feature-for-user-login-system  # Too long
```

## Essential Commands

### Initial Setup

```bash
# Clone repository
git clone git@github.com:org/repo.git
cd repo

# Configure user (if not global)
git config user.name "Your Name"
git config user.email "your@email.com"
```

### Daily Workflow

```bash
# Update main/develop
git checkout main
git pull origin main

# Create branch
git checkout -b feature/my-feature

# Work...
git add .
git commit -m "feat: add user validation"

# Push
git push -u origin feature/my-feature

# Create PR via GitHub/GitLab
```

### Sync with Main

```bash
# Option 1: Rebase (linear history)
git checkout feature/my-feature
git fetch origin
git rebase origin/main

# Option 2: Merge (preserves history)
git checkout feature/my-feature
git merge origin/main
```

### Resolve Conflicts

```bash
# During rebase
git rebase origin/main
# ... resolve conflicts in files ...
git add .
git rebase --continue

# During merge
git merge origin/main
# ... resolve conflicts in files ...
git add .
git commit
```

## Commits

### Structure (Conventional Commits)

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

### Examples

```bash
# Simple
git commit -m "feat: add user registration endpoint"

# With body
git commit -m "fix(auth): resolve token expiration issue

The JWT tokens were not being refreshed correctly due to
timezone mismatch. Fixed by using UTC consistently.

Closes #123"
```

### Rules

1. 72 character limit on first line
2. Use imperative mood ("add" not "added")
3. Don't end with period
4. Separate body with blank line

## Pull Requests

### PR Template

```markdown
## Description
Brief description of what was done and why.

## Type of change
- [ ] New feature
- [ ] Bug fix
- [ ] Breaking change
- [ ] Documentation

## How to test
1. Step 1
2. Step 2
3. Verify result

## Checklist
- [ ] Tests passing
- [ ] Code reviewed (self-review)
- [ ] Documentation updated
- [ ] No breaking changes (or documented)
```

### Best Practices

```bash
# Small and focused PRs
# - Easier to review
# - Less risk of conflicts
# - Faster deployment

# Keep PR updated
git fetch origin
git rebase origin/main
git push --force-with-lease  # Safe for personal branches
```

## Common Operations

### Undo Changes

```bash
# Discard unstaged changes
git checkout -- <file>
git restore <file>  # Git 2.23+

# Unstage file
git reset HEAD <file>
git restore --staged <file>  # Git 2.23+

# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Revert already pushed commit
git revert <commit-hash>
```

### Stash

```bash
# Temporarily save changes
git stash
git stash save "WIP: feature description"

# List stashes
git stash list

# Apply stash
git stash pop        # Apply and remove
git stash apply      # Apply and keep

# Discard stash
git stash drop
```

### Interactive Rebase

```bash
# Reorganize last 3 commits
git rebase -i HEAD~3

# Editor opens with:
pick abc123 First commit
pick def456 Second commit
pick ghi789 Third commit

# Available commands:
# pick   = use commit
# reword = edit message
# squash = merge with previous
# drop   = remove commit
```

### Cherry Pick

```bash
# Copy specific commit to current branch
git cherry-pick <commit-hash>

# Cherry pick without automatic commit
git cherry-pick -n <commit-hash>
```

## Branch Protection

### Recommended Rules (main)

```yaml
# GitHub Branch Protection
- Require pull request reviews: 1+
- Require status checks to pass
- Require branches to be up to date
- Include administrators: false
- Restrict who can push: maintainers only
- Require signed commits: optional
```

## Tags and Releases

```bash
# Create tag
git tag v1.0.0
git tag -a v1.0.0 -m "Release version 1.0.0"

# Push tags
git push origin v1.0.0
git push origin --tags

# List tags
git tag -l "v1.*"

# Delete tag
git tag -d v1.0.0
git push origin --delete v1.0.0
```

## .gitignore

```gitignore
# Dependencies
node_modules/
vendor/
.venv/

# Build
dist/
build/
*.pyc
__pycache__/

# IDE
.idea/
.vscode/
*.swp

# Environment
.env
.env.local
*.local

# Logs
*.log
logs/

# OS
.DS_Store
Thumbs.db

# Secrets (NEVER commit)
*.pem
*.key
credentials.json
```

## Hooks (pre-commit)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
        
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v2.42.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

## Git Checklist

### Before Committing
- [ ] `git diff` to review changes
- [ ] Tests passing locally
- [ ] No sensitive files
- [ ] Clear commit message

### Before Push
- [ ] Branch updated with main
- [ ] Commits organized (squash if needed)
- [ ] CI passing locally

### Before Merge
- [ ] PR approved
- [ ] CI passing
- [ ] Conflicts resolved
- [ ] Branch updated
