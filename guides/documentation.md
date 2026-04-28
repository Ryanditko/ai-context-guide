# Documentation

Guide for writing clear, useful, and maintainable documentation.

## Types of Documentation

### 1. README
Project entry point.

```markdown
# Project Name

Concise description of what the project does (1-2 sentences).

## Quick Start

\`\`\`bash
# Installation
npm install

# Run
npm start
\`\`\`

## Features

- Feature 1
- Feature 2
- Feature 3

## Documentation

- [Installation Guide](docs/installation.md)
- [API Reference](docs/api.md)
- [Contributing](CONTRIBUTING.md)

## Tech Stack

- Node.js 20+
- PostgreSQL 15
- Redis 7

## License

MIT
```

### 2. API Documentation

```markdown
## POST /users

Creates a new user.

### Request

\`\`\`json
{
  "name": "string (required, 1-100 chars)",
  "email": "string (required, valid email)",
  "role": "string (optional, default: 'user')"
}
\`\`\`

### Response

**201 Created**
\`\`\`json
{
  "id": "uuid",
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user",
  "created_at": "2024-01-15T10:30:00Z"
}
\`\`\`

**400 Bad Request**
\`\`\`json
{
  "error": "validation_error",
  "details": [
    {"field": "email", "message": "Invalid email format"}
  ]
}
\`\`\`

### Example

\`\`\`bash
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"name": "John", "email": "john@example.com"}'
\`\`\`
```

### 3. Code Documentation

```python
def calculate_discount(
    original_price: float,
    discount_percent: float,
    max_discount: float | None = None
) -> float:
    """
    Calculate the final price after applying a percentage discount.
    
    Args:
        original_price: The original price before discount.
        discount_percent: Discount percentage (0-100).
        max_discount: Optional maximum discount amount cap.
    
    Returns:
        The final price after discount.
    
    Raises:
        ValueError: If discount_percent is not between 0 and 100.
    
    Example:
        >>> calculate_discount(100.0, 20.0)
        80.0
        >>> calculate_discount(100.0, 50.0, max_discount=30.0)
        70.0
    """
    if not 0 <= discount_percent <= 100:
        raise ValueError("Discount must be between 0 and 100")
    
    discount = original_price * (discount_percent / 100)
    
    if max_discount is not None:
        discount = min(discount, max_discount)
    
    return original_price - discount
```

### 4. ADR (Architecture Decision Records)

```markdown
# ADR-001: Use PostgreSQL as primary database

## Status
Accepted

## Context
We need to choose a database for the new payments service.
Requirements: ACID compliance, JSON support, good performance for complex queries.

## Decision
We will use PostgreSQL 15.

## Consequences

### Positive
- Native ACID compliance
- Excellent JSON support (JSONB)
- Wide ecosystem and community
- Team already has experience

### Negative
- Horizontal scaling more complex than NoSQL
- Requires more tuning for high performance

## Alternatives Considered

### MongoDB
- Rejected: Lack of multi-document transactions was a blocker

### MySQL
- Rejected: JSON support inferior to PostgreSQL
```

## Principles of Good Documentation

### 1. Clear audience
```markdown
# Wrong: Mixed levels
This module uses dependency injection for...
To install, run: npm install

# Right: Sections for each audience
## For Users
How to install and use the project.

## For Developers
Architecture and technical decisions.
```

### 2. Practical examples
```markdown
# Wrong: Description only
The function accepts a callback.

# Right: Description + example
The function accepts a callback that is executed after completion.

\`\`\`javascript
fetchData(url, (error, data) => {
  if (error) {
    console.error('Failed:', error);
    return;
  }
  console.log('Data:', data);
});
\`\`\`
```

### 3. Keep updated
```markdown
# Strategies to keep docs updated:
1. Docs alongside code (docstrings, comments)
2. CI that validates code examples
3. Docs review in PR checklist
4. Documentation owner defined
```

### 4. Consistent structure
```markdown
# Template for functions/methods
## Name
## Description
## Parameters
## Return
## Exceptions
## Example
```

## Comments in Code

### When to comment
```python
# Good: Explains the "why"
# We use retry here because the external API has known
# instability during deploys (usually < 30s)
for attempt in range(3):
    try:
        return api.call()
    except TimeoutError:
        time.sleep(10)

# Good: Documents non-obvious decision
# We sort by ID instead of date to guarantee
# consistency when multiple items have the same timestamp
items.sort(key=lambda x: x.id)
```

### When not to comment
```python
# Bad: Obvious from the code
# Increment the counter
counter += 1

# Bad: Outdated
# Send welcome email
def send_notification(user):  # Now sends push too
    ...

# Bad: Commented code
# old_implementation()
new_implementation()
```

## Tools

### Doc generation
- **Sphinx** (Python)
- **JSDoc** (JavaScript)
- **Swagger/OpenAPI** (APIs)
- **MkDocs** (Markdown)

### Validation
```yaml
# CI to validate docs
- name: Check links
  run: markdown-link-check **/*.md

- name: Lint markdown
  run: markdownlint **/*.md

- name: Validate OpenAPI
  run: openapi-generator validate -i openapi.yaml
```

## Documentation Checklist

### New Project
- [ ] README with quick start
- [ ] Requirements and installation
- [ ] Environment setup
- [ ] How to run tests
- [ ] How to contribute

### New Feature
- [ ] API documentation (if applicable)
- [ ] Usage examples
- [ ] Docstrings updated
- [ ] README updated (if necessary)

### Breaking Change
- [ ] Changelog updated
- [ ] Migration guide
- [ ] Major version bumped
