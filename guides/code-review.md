# Code Review

Guide for conducting and participating in effective code reviews.

## Code Review Goals

1. **Quality**: Identify bugs, logic flaws, and edge cases
2. **Knowledge**: Spread code knowledge across the team
3. **Consistency**: Maintain project standards and conventions
4. **Mentoring**: Opportunity for mutual learning

## For the Author (PR Creator)

### Preparing the PR

```markdown
## Checklist before opening PR
- [ ] Code compiles/passes lint without errors
- [ ] Tests passing locally
- [ ] Self-review completed
- [ ] Clear description of what was done and why
- [ ] PR with appropriate size (< 400 lines ideally)
```

### Writing a good description

```markdown
## Title
feat(auth): implement password reset flow

## Description
### What was done
- POST /auth/reset-password endpoint
- Email with reset link (expires in 1h)
- Token validation and password update

### Why
Users had no way to recover access when they forgot their password.

### How to test
1. POST /auth/forgot-password with valid email
2. Check received email
3. Click link and set new password

### Screenshots (if applicable)
[flow image]

### Checklist
- [x] Unit tests
- [x] Integration tests
- [x] Documentation updated
```

### Responding to feedback

- Thank the feedback, even when disagreeing
- Respond to all comments (resolve or explain)
- Make separate commits for fixes (easier re-review)
- Don't take criticism personally

## For the Reviewer

### Mindset

- Assume good intent from the author
- Be constructive, not destructive
- Suggest, don't order
- Focus on the code, not the person

### What to check

#### 1. Functionality
```
- Does the code do what it proposes?
- Are edge cases handled?
- Are there potential bugs or race conditions?
- Are errors handled appropriately?
```

#### 2. Design
```
- Is the code in the right place (correct layer)?
- Are responsibilities well separated?
- Is there duplication that could be abstracted?
- Is the solution simple or over-engineered?
```

#### 3. Readability
```
- Are names clear and descriptive?
- Are functions appropriately sized?
- Is the logic easy to follow?
- Do comments explain the "why" (not the "what")?
```

#### 4. Tests
```
- Do tests cover the happy path?
- Do tests cover edge cases and errors?
- Are tests readable and maintainable?
- Are assertions clear?
```

#### 5. Security
```
- Are inputs validated?
- Is sensitive data protected?
- Is authentication/authorization correct?
- No SQL injection, XSS, etc?
```

### Comment Types

Use prefixes to clarify the nature of the comment:

| Prefix | Meaning |
|--------|---------|
| `[blocking]` | Must be resolved before merge |
| `[suggestion]` | Optional improvement |
| `[question]` | Need to understand better |
| `[nit]` | Nitpick, very minor |
| `[praise]` | Positive recognition |

### Comment Examples

```markdown
# Bad (vague, no context)
"This is wrong"
"I don't like it"
"Refactor this"

# Good (specific, constructive)
"[blocking] This loop can cause N+1 queries. 
Consider using a JOIN or batch loading."

"[suggestion] Could we extract this logic to a helper? 
Would make it easier to reuse and test."

"[question] Why did we choose polling instead of websockets here?"

"[nit] Typo: 'recieve' → 'receive'"

"[praise] Excellent use of pattern matching here! 
Much more readable than the previous version."
```

### Approving PRs

```markdown
# Approval levels
✅ LGTM (Looks Good To Me) - Approved without reservations
🔄 Approve with comments - Approved, but consider comments
⏸️ Request changes - Needs changes before merge
```

## Best Practices

### PR Size
```
< 200 lines  → Ideal
200-400      → Acceptable
400-800      → Hard to review
> 800        → Split into smaller PRs
```

### Response time
- First response: < 4 business hours
- Complete review: < 24 hours
- Don't let PRs sit for days

### Avoid
- Bikeshedding (endless debates about trivialities)
- Rewriting the author's code in comments
- Blocking for personal style preference
- Rubber stamping (approving without actually reviewing)

## Automation

### Automate what you can
```yaml
# What to leave for CI/CD
- Linting and formatting
- Automated tests
- Static security analysis
- Code coverage

# What needs a human
- Business logic
- Design and architecture
- Readability
- Product context
```

## Health Metrics

| Metric | Target |
|--------|--------|
| Time to first review | < 4h |
| Time to merge | < 48h |
| Average PR size | < 300 lines |
| Rework rate | < 20% |
| Comments per PR | 2-10 |
