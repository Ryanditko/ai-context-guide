# API Design

Guide for designing consistent and intuitive REST APIs.

## Fundamental Principles

### 1. Consistency
Maintain consistent patterns throughout the API.

### 2. Predictability
Developers should be able to guess how endpoints work.

### 3. Simplicity
Start simple, add complexity when necessary.

## URLs and Resources

### Naming Convention

```
# ✅ Good: Plural nouns, lowercase, kebab-case
GET    /users
GET    /users/{id}
GET    /users/{id}/orders
GET    /order-items

# ❌ Bad: Verbs, singular, camelCase
GET    /getUsers
GET    /user/{id}
POST   /createUser
GET    /orderItems
```

### Resource Hierarchy

```
# Nested resources (when it makes sense)
GET    /users/{user_id}/orders           # User's orders
GET    /users/{user_id}/orders/{order_id} # Specific order

# Avoid more than 2 levels of nesting
# ❌ Bad
GET    /users/{id}/orders/{id}/items/{id}/reviews

# ✅ Good: Use query params or independent resources
GET    /order-items/{id}/reviews
GET    /reviews?order_item_id={id}
```

### Query Parameters

```
# Filters
GET /orders?status=pending&created_after=2024-01-01

# Sorting
GET /orders?sort=created_at:desc

# Pagination
GET /orders?page=2&per_page=20
GET /orders?cursor=abc123&limit=20

# Specific fields
GET /users/123?fields=id,name,email

# Relationship expansion
GET /orders/123?expand=user,items
```

## HTTP Methods

| Method | Usage | Idempotent | Safe |
|--------|-------|------------|------|
| GET | Read resource(s) | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace resource | Yes | No |
| PATCH | Partial update | No* | No |
| DELETE | Remove resource | Yes | No |

```
# Basic CRUD
GET    /users          # List
POST   /users          # Create
GET    /users/{id}     # Read one
PUT    /users/{id}     # Replace completely
PATCH  /users/{id}     # Update partially
DELETE /users/{id}     # Remove

# Actions that are not CRUD
POST   /orders/{id}/cancel      # Specific action
POST   /users/{id}/send-verification-email
```

## Status Codes

### Success (2xx)

```
200 OK              # Successful GET, PUT, PATCH
201 Created         # POST that created resource
202 Accepted        # Action accepted, async processing
204 No Content      # Successful DELETE, no body
```

### Redirect (3xx)

```
301 Moved Permanently  # Resource permanently moved
304 Not Modified       # Cache still valid
```

### Client Error (4xx)

```
400 Bad Request        # Validation error
401 Unauthorized       # Not authenticated
403 Forbidden          # Not authorized
404 Not Found          # Resource doesn't exist
409 Conflict           # State conflict
422 Unprocessable      # Invalid semantics
429 Too Many Requests  # Rate limit exceeded
```

### Server Error (5xx)

```
500 Internal Error     # Unexpected error
502 Bad Gateway        # Upstream service error
503 Service Unavailable # Service temporarily unavailable
504 Gateway Timeout    # Upstream service timeout
```

## Request/Response Format

### Request Body

```json
// POST /users
{
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user"
}

// PATCH /users/123 (partial)
{
  "email": "newemail@example.com"
}
```

### Response Body

```json
// Single resource
{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2024-01-15T10:30:00Z"
}

// List with pagination
{
  "data": [
    {"id": "1", "name": "Item 1"},
    {"id": "2", "name": "Item 2"}
  ],
  "pagination": {
    "total": 100,
    "page": 1,
    "per_page": 20,
    "total_pages": 5
  }
}

// Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data",
    "details": [
      {"field": "email", "message": "Invalid format"}
    ]
  }
}
```

## Versioning

### URL Path (recommended)
```
GET /v1/users
GET /v2/users
```

### Header
```
GET /users
Accept: application/vnd.api+json; version=1
```

### Query Parameter
```
GET /users?version=1
```

## Authentication

### Bearer Token (JWT)
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

### API Key
```
# Header
X-API-Key: your-api-key

# Query param (less secure)
GET /users?api_key=your-api-key
```

## Rate Limiting

### Response Headers
```
X-RateLimit-Limit: 1000        # Limit per window
X-RateLimit-Remaining: 999     # Remaining
X-RateLimit-Reset: 1640000000  # Reset timestamp
Retry-After: 60                # Seconds until retry (when 429)
```

### 429 Response
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests",
    "retry_after": 60
  }
}
```

## HATEOAS (Hypermedia)

```json
{
  "id": "123",
  "status": "pending",
  "total": 99.99,
  "_links": {
    "self": {"href": "/orders/123"},
    "cancel": {"href": "/orders/123/cancel", "method": "POST"},
    "pay": {"href": "/orders/123/pay", "method": "POST"},
    "items": {"href": "/orders/123/items"}
  }
}
```

## Documentation (OpenAPI)

```yaml
openapi: 3.0.0
info:
  title: My API
  version: 1.0.0

paths:
  /users:
    get:
      summary: List users
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [active, inactive]
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
    post:
      summary: Create user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: Created

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        email:
          type: string
          format: email
```

## API Design Checklist

### URLs
- [ ] Plural nouns
- [ ] Lowercase with kebab-case
- [ ] Maximum 2 levels of nesting
- [ ] Versioning defined

### Methods and Status
- [ ] Correct HTTP methods
- [ ] Appropriate status codes
- [ ] Idempotency respected

### Request/Response
- [ ] Consistent JSON format
- [ ] Pagination for lists
- [ ] Standardized error format
- [ ] Timestamps in ISO 8601

### Security
- [ ] Authentication implemented
- [ ] Rate limiting active
- [ ] HTTPS mandatory
- [ ] CORS configured

### Documentation
- [ ] OpenAPI/Swagger spec
- [ ] Request/response examples
- [ ] Error codes documented
