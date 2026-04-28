# AI Prompts for Development

Guide for crafting effective prompts when working with AI coding assistants.

## Prompt Engineering Principles

### The CLEAR Framework

| Principle | Description | Example |
|-----------|-------------|---------|
| **C**ontext | Provide relevant background | "In our Python FastAPI microservice..." |
| **L**anguage | Specify tech stack | "Using TypeScript, React 18, Tailwind" |
| **E**xpectations | Define output format | "Respond with only the code, no explanations" |
| **A**ction | Clear task description | "Refactor this function to use async/await" |
| **R**estrictions | State constraints | "Don't use external libraries" |

---

## Code Generation Prompts

### Creating New Code

```markdown
# Template: New Feature Implementation

Context: [Brief description of the system/module]
Tech Stack: [Language, framework, libraries]
Task: [What needs to be built]

Requirements:
- [Requirement 1]
- [Requirement 2]
- [Requirement 3]

Constraints:
- [Constraint 1]
- [Constraint 2]

Expected output:
- [Files to create]
- [Structure expectations]
```

**Example:**
```markdown
Context: E-commerce backend service handling order processing
Tech Stack: Python 3.12, FastAPI, SQLAlchemy, PostgreSQL
Task: Create an endpoint for order cancellation

Requirements:
- Only pending or confirmed orders can be cancelled
- Must refund payment if already charged
- Must release inventory reservations
- Must send cancellation email to customer

Constraints:
- Follow existing project structure (Clean Architecture)
- Use existing PaymentGateway and InventoryService
- Include proper error handling and logging

Expected output:
- Use case class in application/use_cases/
- Controller method in adapters/controllers/
- Integration with existing services
```

### Refactoring Existing Code

```markdown
# Template: Refactoring Request

Current code:
[Paste the code to refactor]

Issues with current code:
- [Issue 1]
- [Issue 2]

Desired improvements:
- [Improvement 1]
- [Improvement 2]

Constraints:
- Maintain backward compatibility
- Keep same public interface
- [Other constraints]
```

**Example:**
```markdown
Current code:
```python
def process_order(order_id, user_id, items, discount_code, shipping_type):
    # 200 lines of nested conditions
    ...
```

Issues with current code:
- Function too long (200+ lines)
- Too many parameters (5)
- Mixed responsibilities (validation, pricing, persistence)
- Hard to test

Desired improvements:
- Split into smaller, focused functions
- Group related parameters into objects
- Separate validation from business logic
- Make it testable

Constraints:
- Keep same function signature for now (will deprecate later)
- Don't change database schema
```

### Bug Fixing

```markdown
# Template: Bug Fix Request

Bug description: [What's happening]
Expected behavior: [What should happen]
Actual behavior: [What's happening instead]

Relevant code:
[Paste the code]

Error message (if any):
[Paste error/stack trace]

Steps to reproduce:
1. [Step 1]
2. [Step 2]

Environment:
- [Language version]
- [Framework version]
- [OS if relevant]
```

---

## Code Review Prompts

### Reviewing Code Quality

```markdown
# Template: Code Review

Review the following code for:
- [ ] Potential bugs
- [ ] Security vulnerabilities
- [ ] Performance issues
- [ ] Code style violations
- [ ] Missing error handling
- [ ] Test coverage gaps

Code to review:
[Paste code]

Context: [What this code does]
Language/Framework: [Tech stack]

Provide feedback in this format:
- Severity: [Critical/Major/Minor/Suggestion]
- Location: [Line number or function name]
- Issue: [Description]
- Suggestion: [How to fix]
```

### Security Review

```markdown
# Template: Security Review

Analyze the following code for security vulnerabilities:

Code:
[Paste code]

Check for:
- SQL injection
- XSS vulnerabilities
- Authentication/authorization issues
- Sensitive data exposure
- Input validation gaps
- Insecure dependencies

Severity classification:
- Critical: Exploitable vulnerability
- High: Potential for data breach
- Medium: Defense in depth issue
- Low: Best practice violation
```

---

## Testing Prompts

### Generating Unit Tests

```markdown
# Template: Unit Test Generation

Generate unit tests for the following code:

Code to test:
[Paste code]

Test framework: [pytest/jest/junit/etc]
Mocking library: [if applicable]

Requirements:
- Test happy path
- Test edge cases
- Test error conditions
- Use descriptive test names
- Follow AAA pattern (Arrange-Act-Assert)

Specific scenarios to cover:
- [Scenario 1]
- [Scenario 2]
```

**Example:**
```markdown
Generate unit tests for the following code:

```python
class PricingService:
    def calculate_total(self, items: list[OrderItem], discount: Discount = None) -> Money:
        subtotal = sum(item.subtotal for item in items)
        if discount:
            subtotal = discount.apply(subtotal)
        return subtotal
```

Test framework: pytest
Mocking library: pytest-mock

Requirements:
- Test with empty items list
- Test with single item
- Test with multiple items
- Test with percentage discount
- Test with fixed discount
- Test discount larger than subtotal
- Test currency mismatch error
```

### Integration Test Scenarios

```markdown
# Template: Integration Test Scenarios

Generate integration test scenarios for:

Feature: [Feature name]
Components involved: [List of components]

Test environment:
- Database: [Type]
- External services: [Mock/Real]
- Message queues: [If applicable]

Scenarios needed:
1. Happy path: [Description]
2. Error case: [Description]
3. Edge case: [Description]

Include:
- Setup steps
- Test execution
- Assertions
- Cleanup
```

---

## Documentation Prompts

### Code Documentation

```markdown
# Template: Documentation Generation

Generate documentation for:

Code:
[Paste code]

Documentation type:
- [ ] Function/method docstrings
- [ ] Class documentation
- [ ] Module documentation
- [ ] API documentation
- [ ] README section

Style guide: [Google/NumPy/Sphinx/JSDoc/etc]

Include:
- Description
- Parameters with types
- Return values
- Exceptions/errors
- Usage examples
```

### README Generation

```markdown
# Template: README Generation

Generate a README for:

Project name: [Name]
Project type: [Library/CLI/API/Application]
Main language: [Language]

Include sections:
- [ ] Project description
- [ ] Features
- [ ] Installation
- [ ] Quick start
- [ ] Configuration
- [ ] Usage examples
- [ ] API reference
- [ ] Contributing
- [ ] License

Target audience: [Developers/End users/Both]
```

---

## Architecture Prompts

### Design Review

```markdown
# Template: Architecture Review

Review this architectural design:

Context:
[System description and requirements]

Current/Proposed architecture:
[Describe or diagram]

Concerns:
- Scalability: [Requirements]
- Performance: [Requirements]
- Security: [Requirements]
- Maintainability: [Requirements]

Evaluate:
- Trade-offs of this approach
- Potential issues at scale
- Alternative approaches
- Recommended improvements
```

### Design Decisions

```markdown
# Template: Architecture Decision

Need help deciding between approaches:

Problem: [What needs to be solved]

Option A: [Description]
- Pros: [List]
- Cons: [List]

Option B: [Description]
- Pros: [List]
- Cons: [List]

Constraints:
- [Constraint 1]
- [Constraint 2]

Priorities:
1. [Most important factor]
2. [Second most important]
3. [Third most important]

Please analyze and recommend with justification.
```

---

## Debugging Prompts

### Error Analysis

```markdown
# Template: Error Analysis

Help debug this error:

Error message:
[Full error message and stack trace]

Code causing the error:
[Relevant code]

What I've tried:
- [Attempt 1]
- [Attempt 2]

Environment:
- Language version: [X]
- Framework version: [X]
- OS: [X]

Expected behavior: [What should happen]
Actual behavior: [What's happening]
```

### Performance Investigation

```markdown
# Template: Performance Analysis

Analyze performance of:

Code:
[Paste code]

Current performance:
- Execution time: [X ms/s]
- Memory usage: [X MB]
- Throughput: [X ops/sec]

Target performance:
- Execution time: [X ms/s]
- Memory usage: [X MB]

Workload characteristics:
- Data size: [X]
- Concurrency: [X]
- Frequency: [X]

Identify:
- Bottlenecks
- Optimization opportunities
- Trade-offs
```

---

## Prompt Anti-Patterns

### Avoid These Patterns

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| Vague requests | "Make this better" | Specific improvements needed |
| No context | Code without explanation | Explain purpose and constraints |
| Too broad | "Build me an app" | Break into specific tasks |
| Assuming knowledge | Using project-specific terms | Explain domain concepts |
| No constraints | Any solution acceptable | Specify requirements |

### Bad vs Good Prompts

**Bad:**
```
Fix this code
```

**Good:**
```
Fix the null pointer exception in this function.
The error occurs when `user.address` is None.
The function should return a default address in that case.

Code:
[paste code]

Error:
[paste error]
```

**Bad:**
```
Write tests
```

**Good:**
```
Write unit tests for the `calculateDiscount` function using Jest.
Cover these cases:
1. No discount code provided
2. Valid percentage discount (10%)
3. Expired discount code
4. Minimum purchase not met

Use the AAA pattern and descriptive test names.
```

---

## Context Management

### For Long Conversations

```markdown
# Summary of our work so far:

1. We're building: [Feature/System]
2. Tech stack: [Languages, frameworks]
3. Architecture: [Pattern being used]
4. Files modified: [List]
5. Current task: [What we're doing now]
6. Blockers: [Any issues]

Please continue with: [Next step]
```

### For Complex Projects

```markdown
# Project Context

## Overview
[1-2 sentence description]

## Architecture
[Brief architecture description]

## Current Focus
[Module/feature being worked on]

## Relevant Files
- `path/to/file1.ts` - [Purpose]
- `path/to/file2.ts` - [Purpose]

## Conventions
- [Convention 1]
- [Convention 2]

## Current Task
[Specific task to complete]
```

---

## Response Format Instructions

### When You Want Specific Formats

```markdown
# Code Only
Respond with only the code. No explanations, no markdown formatting.

# Structured Response
Respond in this format:
1. Summary (1 sentence)
2. Changes needed (bullet list)
3. Code (with file paths)
4. Testing notes

# Diff Format
Show changes as a diff:
- Lines to remove
+ Lines to add

# Step by Step
Provide numbered steps. After each step, wait for confirmation before proceeding.
```

---

## Quick Reference Templates

### One-Liner Tasks

```markdown
Convert this JavaScript function to TypeScript with proper types:
[code]
```

```markdown
Add error handling to this function. Throw custom errors for invalid input:
[code]
```

```markdown
Optimize this SQL query for better performance on large datasets:
[query]
```

### Standard Requests

```markdown
Explain what this code does and identify any issues:
[code]
```

```markdown
Suggest 3 ways to improve this code's readability:
[code]
```

```markdown
Add JSDoc comments to all public methods in this class:
[code]
```
