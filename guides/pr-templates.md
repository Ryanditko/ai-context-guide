# PR and Issue Templates

Templates for creating consistent, informative Pull Requests and Issues that enable efficient collaboration and review.

## Pull Request Templates

### Standard Feature PR

```markdown
## Summary

<!-- Brief description of what this PR does (1-2 sentences) -->

## Motivation

<!-- Why is this change needed? Link to issue if applicable -->

Closes #<issue_number>

## Changes

<!-- Bullet points of key changes made -->

- 
- 
- 

## Type of Change

<!-- Check all that apply -->

- [ ] New feature (non-breaking change adding functionality)
- [ ] Bug fix (non-breaking change fixing an issue)
- [ ] Breaking change (fix or feature causing existing functionality to change)
- [ ] Refactoring (no functional changes)
- [ ] Documentation update
- [ ] Performance improvement
- [ ] Test coverage improvement

## How to Test

<!-- Step-by-step instructions to verify the changes -->

1. 
2. 
3. 

## Screenshots/Recordings

<!-- If applicable, add screenshots or recordings -->

## Checklist

<!-- Verify before requesting review -->

- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Tests added/updated for changes
- [ ] Documentation updated if needed
- [ ] No new warnings or errors introduced
- [ ] Dependent changes merged and published

## Additional Notes

<!-- Any other context reviewers should know -->
```

### Bug Fix PR

```markdown
## Bug Description

<!-- Clear description of the bug being fixed -->

## Root Cause

<!-- What was causing the bug? -->

## Solution

<!-- How does this PR fix the bug? -->

## Affected Areas

<!-- What parts of the system are affected? -->

- 
- 

## Regression Risk

<!-- Low/Medium/High - What could break? -->

- **Risk Level**: 
- **Potential Impact**: 

## Verification

### Before Fix
<!-- Describe or show the broken behavior -->

### After Fix
<!-- Describe or show the correct behavior -->

## Test Cases Added

<!-- List new tests that prevent regression -->

- [ ] Test case 1: 
- [ ] Test case 2: 

## Checklist

- [ ] Root cause identified and documented
- [ ] Fix addresses root cause (not just symptoms)
- [ ] Regression tests added
- [ ] Related areas checked for similar issues
- [ ] No new bugs introduced
```

### Refactoring PR

```markdown
## Refactoring Scope

<!-- What is being refactored and why? -->

## Goals

<!-- What improvements does this refactoring achieve? -->

- [ ] Improved readability
- [ ] Better performance
- [ ] Reduced complexity
- [ ] Improved testability
- [ ] Better separation of concerns
- [ ] Removed code duplication
- [ ] Updated deprecated patterns

## Changes Overview

### Before
<!-- Brief description or code snippet of old approach -->

```
// Old approach
```

### After
<!-- Brief description or code snippet of new approach -->

```
// New approach
```

## Metrics (if applicable)

| Metric | Before | After |
|--------|--------|-------|
| Lines of code | | |
| Cyclomatic complexity | | |
| Test coverage | | |
| Build time | | |

## Risk Assessment

- **Behavioral changes**: None expected
- **Breaking changes**: None
- **Rollback plan**: 

## Verification Strategy

<!-- How to verify no behavior changed -->

- [ ] All existing tests pass
- [ ] Manual testing of affected flows
- [ ] Performance benchmarks (if applicable)

## Checklist

- [ ] No functional changes (pure refactoring)
- [ ] All tests pass without modification
- [ ] Code review focused on design improvement
- [ ] Documentation updated if APIs changed
```

### Hotfix PR

```markdown
## 🚨 HOTFIX

**Severity**: Critical / High / Medium
**Affected Environment**: Production / Staging
**Incident Link**: <!-- Link to incident ticket -->

## Problem

<!-- What is broken in production? -->

## Impact

<!-- Who/what is affected? -->

- **Users affected**: 
- **Business impact**: 
- **Started at**: 

## Root Cause

<!-- Quick root cause analysis -->

## Fix

<!-- What does this fix do? -->

## Testing Done

<!-- Fast but thorough verification -->

- [ ] Local testing
- [ ] Staging deployment verified
- [ ] Smoke tests passed

## Rollback Plan

<!-- How to revert if fix fails -->

```bash
# Rollback command
```

## Post-Mortem

- [ ] Incident documented
- [ ] Follow-up tasks created
- [ ] Monitoring added

## Approvals Required

- [ ] Tech Lead
- [ ] On-call engineer
- [ ] Product (if user-facing)
```

### Database Migration PR

```markdown
## Migration Overview

**Migration Name**: 
**Reversible**: Yes / No

## Schema Changes

<!-- Describe database schema modifications -->

### Tables Affected

| Table | Change Type | Description |
|-------|-------------|-------------|
| | CREATE / ALTER / DROP | |

### New Columns

| Table | Column | Type | Nullable | Default |
|-------|--------|------|----------|---------|
| | | | | |

### Indexes

| Table | Index Name | Columns | Type |
|-------|------------|---------|------|
| | | | |

## Migration Scripts

### Up Migration

```sql
-- Forward migration
```

### Down Migration

```sql
-- Rollback migration
```

## Data Migration

<!-- If migrating existing data -->

- **Records affected**: ~
- **Estimated duration**: 
- **Batching strategy**: 

## Deployment Plan

1. [ ] Backup database
2. [ ] Deploy application with backward compatibility
3. [ ] Run migration
4. [ ] Verify data integrity
5. [ ] Deploy final application version
6. [ ] Remove backward compatibility code

## Rollback Plan

<!-- Steps to revert if issues occur -->

1. 
2. 

## Checklist

- [ ] Migration tested on copy of production data
- [ ] Rollback tested
- [ ] Performance impact assessed
- [ ] Application handles both old and new schema (during transition)
- [ ] No data loss scenarios
- [ ] Indexes added for new query patterns
```

### API Change PR

```markdown
## API Change Summary

**Type**: New Endpoint / Breaking Change / Deprecation / Enhancement
**Version**: v1 → v2 (if applicable)

## Endpoint Changes

### New Endpoints

| Method | Path | Description |
|--------|------|-------------|
| | | |

### Modified Endpoints

| Endpoint | Change | Migration Path |
|----------|--------|----------------|
| | | |

### Deprecated Endpoints

| Endpoint | Deprecated Date | Removal Date | Replacement |
|----------|-----------------|--------------|-------------|
| | | | |

## Request/Response Changes

### Before

```json
{
  // Old format
}
```

### After

```json
{
  // New format
}
```

## Breaking Changes

<!-- List any breaking changes and migration instructions -->

- 
- 

## Client Migration Guide

<!-- How should API consumers update their code? -->

```
// Before
client.oldMethod()

// After
client.newMethod()
```

## Documentation

- [ ] OpenAPI/Swagger spec updated
- [ ] API documentation updated
- [ ] Changelog updated
- [ ] Client library updated (if maintained)

## Testing

- [ ] Contract tests pass
- [ ] Integration tests updated
- [ ] Backward compatibility verified
- [ ] Load testing (if significant change)

## Rollout Plan

- [ ] Feature flag (if applicable)
- [ ] Gradual rollout percentage
- [ ] Monitoring dashboards ready
- [ ] Alerts configured
```

---

## Issue Templates

### Bug Report

```markdown
## Bug Report

### Environment

- **App Version**: 
- **OS**: 
- **Browser** (if applicable): 
- **Device** (if applicable): 

### Description

<!-- Clear, concise description of the bug -->

### Steps to Reproduce

1. 
2. 
3. 
4. 

### Expected Behavior

<!-- What should happen -->

### Actual Behavior

<!-- What actually happens -->

### Screenshots/Logs

<!-- Add screenshots, error messages, or logs -->

```
// Error logs here
```

### Frequency

- [ ] Always reproducible
- [ ] Intermittent
- [ ] Happened once

### Workaround

<!-- If known, describe any workaround -->

### Additional Context

<!-- Any other relevant information -->
```

### Feature Request

```markdown
## Feature Request

### Problem Statement

<!-- What problem does this feature solve? -->

As a [type of user], I want [goal] so that [benefit].

### Proposed Solution

<!-- Describe the solution you'd like -->

### User Stories

<!-- Break down into specific user stories -->

- [ ] As a user, I can...
- [ ] As a user, I can...

### Acceptance Criteria

<!-- When is this feature "done"? -->

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

### Alternatives Considered

<!-- What other solutions did you consider? -->

1. **Alternative A**: 
   - Pros: 
   - Cons: 

2. **Alternative B**: 
   - Pros: 
   - Cons: 

### Mockups/Designs

<!-- If applicable, add wireframes or designs -->

### Technical Considerations

<!-- Any technical constraints or considerations -->

- Dependencies: 
- Performance impact: 
- Security considerations: 

### Priority

- [ ] Critical (blocking users)
- [ ] High (significant impact)
- [ ] Medium (nice to have)
- [ ] Low (future consideration)

### Additional Context

<!-- Any other relevant information -->
```

### Technical Debt

```markdown
## Technical Debt

### Area

<!-- What part of the codebase is affected? -->

- **Module/Component**: 
- **Files**: 

### Current State

<!-- Describe the current problematic state -->

### Problems Caused

<!-- What issues does this tech debt cause? -->

- [ ] Slows development
- [ ] Causes bugs
- [ ] Performance issues
- [ ] Security vulnerabilities
- [ ] Maintenance burden
- [ ] Testing difficulties

### Proposed Improvement

<!-- How should this be fixed? -->

### Effort Estimate

- **Size**: Small / Medium / Large / XL
- **Estimated time**: 
- **Risk level**: Low / Medium / High

### Benefits

<!-- What improvements will this bring? -->

- 
- 

### Dependencies

<!-- What needs to happen first? -->

- 
- 

### Definition of Done

- [ ] Code refactored
- [ ] Tests updated/added
- [ ] Documentation updated
- [ ] No regression in functionality
- [ ] Performance maintained or improved
```

### Spike/Research

```markdown
## Spike: [Topic]

### Objective

<!-- What question are we trying to answer? -->

### Background

<!-- Context and why this research is needed -->

### Questions to Answer

1. 
2. 
3. 

### Time Box

- **Duration**: 
- **Due date**: 

### Approach

<!-- How will you research this? -->

- [ ] Documentation review
- [ ] Proof of concept
- [ ] Vendor evaluation
- [ ] Performance testing
- [ ] Security assessment

### Success Criteria

<!-- What output is expected? -->

- [ ] Decision document
- [ ] Proof of concept code
- [ ] Recommendation with pros/cons
- [ ] Estimate for implementation

### Findings

<!-- To be filled after research -->

### Recommendation

<!-- Final recommendation based on findings -->

### Next Steps

<!-- What should happen after this spike? -->

- [ ] 
- [ ] 
```

### Incident Report

```markdown
## Incident Report

### Incident ID: INC-XXXX
### Date: YYYY-MM-DD
### Severity: Critical / High / Medium / Low

---

## Summary

<!-- One paragraph summary of what happened -->

## Timeline

| Time (UTC) | Event |
|------------|-------|
| HH:MM | Issue first detected |
| HH:MM | Alert triggered |
| HH:MM | Investigation started |
| HH:MM | Root cause identified |
| HH:MM | Fix deployed |
| HH:MM | Service restored |

## Impact

- **Duration**: 
- **Users affected**: 
- **Revenue impact**: 
- **SLA impact**: 

## Root Cause

<!-- Technical explanation of what caused the incident -->

## Contributing Factors

<!-- What conditions led to this happening? -->

1. 
2. 

## Resolution

<!-- How was the incident resolved? -->

## Detection

- **How detected**: Alert / Customer report / Monitoring / Manual check
- **Time to detect**: 
- **Detection gaps**: 

## Action Items

### Immediate (< 1 week)

- [ ] [P0] Action item 1 - @owner
- [ ] [P0] Action item 2 - @owner

### Short-term (< 1 month)

- [ ] [P1] Action item 3 - @owner
- [ ] [P1] Action item 4 - @owner

### Long-term (< 1 quarter)

- [ ] [P2] Action item 5 - @owner

## Lessons Learned

### What went well

- 
- 

### What could be improved

- 
- 

## Appendix

### Relevant Links

- Monitoring dashboard: 
- Logs: 
- Related incidents: 

### Graphs/Screenshots

<!-- Include relevant monitoring graphs -->
```

---

## PR Review Checklist

Use this checklist when reviewing PRs:

```markdown
## Review Checklist

### Code Quality
- [ ] Code is readable and self-documenting
- [ ] No unnecessary complexity
- [ ] Follows project conventions
- [ ] No code duplication

### Functionality
- [ ] Solves the stated problem
- [ ] Edge cases handled
- [ ] Error handling appropriate
- [ ] No obvious bugs

### Testing
- [ ] Tests cover happy path
- [ ] Tests cover edge cases
- [ ] Tests cover error cases
- [ ] Tests are readable and maintainable

### Security
- [ ] No hardcoded secrets
- [ ] Input validation present
- [ ] Authentication/authorization correct
- [ ] No sensitive data exposure

### Performance
- [ ] No N+1 queries
- [ ] No unnecessary computations
- [ ] Appropriate caching
- [ ] Resource cleanup handled

### Documentation
- [ ] Public APIs documented
- [ ] Complex logic explained
- [ ] README updated if needed
- [ ] Changelog updated if needed
```

---

## Best Practices

### PR Titles

```
# Format: <type>(<scope>): <description>

# Examples:
feat(auth): add OAuth2 login support
fix(orders): correct tax calculation for EU customers
refactor(api): extract validation middleware
docs(readme): add deployment instructions
test(users): add integration tests for signup flow
perf(search): add caching for product queries
chore(deps): upgrade React to v18
```

### PR Size Guidelines

| Lines Changed | Category | Review Time |
|---------------|----------|-------------|
| < 50 | Tiny | ~5 min |
| 50-200 | Small | ~15 min |
| 200-400 | Medium | ~30 min |
| 400-800 | Large | ~1 hour |
| > 800 | Too Large | Split it! |

### When to Split PRs

- Different logical changes
- Feature + refactoring
- Multiple unrelated fixes
- Large migrations
- Changes to multiple systems
