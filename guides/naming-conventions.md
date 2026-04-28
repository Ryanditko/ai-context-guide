# Naming Conventions

Guide for consistent, meaningful naming across code, files, and resources.

## Core Principles

### Names Should Be

1. **Intention-Revealing**: Name describes purpose, not implementation
2. **Pronounceable**: Can be spoken in conversation
3. **Searchable**: Unique enough to find with search
4. **Consistent**: Same concept, same name everywhere
5. **Appropriate Length**: Long enough to be clear, short enough to be readable

### The 3C Rule

```
Clear    → Immediately understandable
Concise  → No unnecessary words
Consistent → Same pattern throughout
```

---

## Variables

### General Rules

| Rule | Bad | Good |
|------|-----|------|
| Avoid abbreviations | `usr`, `cnt`, `val` | `user`, `count`, `value` |
| Avoid single letters | `x`, `n`, `t` | `index`, `count`, `timeout` |
| Be specific | `data`, `info`, `item` | `userData`, `orderInfo`, `productItem` |
| Avoid noise words | `theUser`, `aProduct` | `user`, `product` |
| Use domain terms | `d`, `elapsed` | `deliveryDate`, `elapsedTimeMs` |

### Booleans

```javascript
// ✅ Good: Question form, positive assertion
const isActive = true;
const hasPermission = false;
const canEdit = true;
const shouldRetry = false;
const wasProcessed = true;

// ❌ Bad: Negative or unclear
const notActive = false;      // Use: isActive = true
const isNotValid = true;      // Use: isInvalid = true or isValid = false
const flag = true;            // What flag?
const status = true;          // Boolean status?
```

### Collections

```javascript
// ✅ Good: Plural nouns
const users = [];
const orderItems = [];
const activeSubscriptions = [];

// ✅ Good: Descriptive suffixes for maps
const userById = new Map();
const productsByCategory = {};
const pricePerUnit = {};

// ❌ Bad: Unclear
const list = [];              // List of what?
const data = [];              // What data?
const userList = [];          // Redundant "List"
```

### Numbers

```javascript
// ✅ Good: Include unit or purpose
const timeoutMs = 5000;
const maxRetryCount = 3;
const priceInCents = 9999;
const distanceKm = 42.195;
const weightKg = 75.5;
const ageYears = 25;

// ❌ Bad: Ambiguous units
const timeout = 5000;         // Seconds? Milliseconds?
const price = 99.99;          // Dollars? Cents?
const distance = 42;          // Miles? Kilometers?
```

### Dates and Times

```javascript
// ✅ Good: Clear purpose
const createdAt = new Date();
const updatedAt = new Date();
const expiresAt = new Date();
const startDate = new Date();
const endDate = new Date();
const scheduledFor = new Date();

// ❌ Bad: Ambiguous
const date = new Date();      // What date?
const time = new Date();      // What time?
const timestamp = 1234567890; // Created? Updated? When?
```

---

## Functions and Methods

### Naming Patterns

| Type | Pattern | Examples |
|------|---------|----------|
| Actions | `verb + noun` | `createUser`, `sendEmail`, `calculateTotal` |
| Getters | `get + noun` | `getUser`, `getOrderTotal`, `getCurrentTime` |
| Setters | `set + noun` | `setStatus`, `setPrice`, `setActiveFlag` |
| Predicates | `is/has/can + adj/noun` | `isValid`, `hasPermission`, `canEdit` |
| Converters | `to + target` | `toString`, `toJson`, `toDollars` |
| Factories | `create/make/build + noun` | `createOrder`, `makeRequest`, `buildQuery` |
| Event handlers | `on/handle + event` | `onClick`, `handleSubmit`, `onUserCreated` |

### Verbs by Operation

```javascript
// CRUD Operations
createUser()    // Create new
getUser()       // Read single
getAllUsers()   // Read many
updateUser()    // Update existing
deleteUser()    // Remove
upsertUser()    // Create or update

// Search & Filter
findUserByEmail()
searchProducts()
filterActiveOrders()
queryCustomers()

// Validation
validateEmail()
checkPermission()
verifyToken()
ensureAuthenticated()

// State Changes
activateAccount()
deactivateUser()
approveRequest()
rejectApplication()
completeOrder()
cancelSubscription()

// Calculations
calculateTotal()
computeDiscount()
sumOrderItems()
countActiveUsers()
```

### Method Length Guideline

```javascript
// ✅ Good: Name describes what, not how
function calculateOrderTotal(order) { ... }
function sendWelcomeEmail(user) { ... }

// ❌ Bad: Name describes implementation
function loopThroughItemsAndSumPrices(order) { ... }
function connectToSmtpAndSendEmail(user) { ... }
```

---

## Classes and Types

### Class Names

```javascript
// ✅ Good: Noun or noun phrase
class User { }
class OrderService { }
class PaymentGateway { }
class EmailNotification { }
class ShoppingCart { }

// ❌ Bad: Verbs, vague names
class ProcessOrder { }        // Use: OrderProcessor
class Manager { }             // Manager of what?
class Helper { }              // Helper for what?
class Utils { }               // Too generic
class Data { }                // What data?
```

### Interface Names

```typescript
// ✅ Good: Describes capability or contract
interface Serializable { }
interface UserRepository { }
interface PaymentProcessor { }
interface EventListener { }

// Style varies by language/team:
interface IUserRepository { }  // C# style
interface UserRepositoryPort { } // Ports & Adapters style
```

### Type Names

```typescript
// ✅ Good: Clear purpose
type UserId = string;
type EmailAddress = string;
type OrderStatus = 'pending' | 'confirmed' | 'shipped';
type Currency = 'USD' | 'EUR' | 'GBP';

// ✅ Good: Descriptive compound types
type CreateUserRequest = { ... };
type OrderSummaryResponse = { ... };
type ValidationResult = { isValid: boolean; errors: string[] };
```

### Enum Names

```typescript
// ✅ Good: Singular name, clear values
enum OrderStatus {
  PENDING = 'pending',
  CONFIRMED = 'confirmed',
  SHIPPED = 'shipped',
  DELIVERED = 'delivered',
  CANCELLED = 'cancelled',
}

enum HttpMethod {
  GET = 'GET',
  POST = 'POST',
  PUT = 'PUT',
  DELETE = 'DELETE',
}

// ❌ Bad: Plural name, unclear values
enum Statuses {
  S1 = 1,
  S2 = 2,
}
```

---

## Files and Directories

### File Naming

| Convention | Style | Example | Common Use |
|------------|-------|---------|------------|
| kebab-case | `lowercase-with-dashes` | `user-service.ts` | Most languages |
| snake_case | `lowercase_with_underscores` | `user_service.py` | Python |
| PascalCase | `CapitalizedWords` | `UserService.java` | Java classes |
| camelCase | `camelCaseWords` | `userService.js` | Some JS projects |

### Directory Structure

```
src/
├── components/          # UI components (PascalCase)
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx
│   │   └── Button.styles.ts
│   └── UserProfile/
│       └── UserProfile.tsx
├── services/            # Business logic (kebab-case)
│   ├── user-service.ts
│   ├── order-service.ts
│   └── payment-service.ts
├── models/              # Data models
│   ├── user.ts
│   ├── order.ts
│   └── product.ts
├── utils/               # Utilities
│   ├── date-utils.ts
│   ├── string-utils.ts
│   └── validation-utils.ts
└── types/               # Type definitions
    ├── api-types.ts
    └── domain-types.ts
```

### Test Files

```
# Pattern: [name].test.[ext] or [name].spec.[ext]

user-service.ts
user-service.test.ts      # Unit tests
user-service.spec.ts      # Alternative convention

# Pattern: __tests__ directory
src/
├── services/
│   └── user-service.ts
└── __tests__/
    └── services/
        └── user-service.test.ts
```

---

## API and Database

### REST Endpoints

```
# Resources: Plural nouns, kebab-case
GET    /users
GET    /users/{id}
POST   /users
PUT    /users/{id}
DELETE /users/{id}

GET    /order-items
GET    /user-profiles
GET    /payment-methods

# Nested resources
GET    /users/{id}/orders
GET    /orders/{id}/items

# Actions (when CRUD doesn't fit)
POST   /orders/{id}/cancel
POST   /users/{id}/reset-password
POST   /payments/{id}/refund
```

### Database Tables and Columns

```sql
-- Tables: Plural, snake_case
CREATE TABLE users ( ... );
CREATE TABLE order_items ( ... );
CREATE TABLE payment_methods ( ... );

-- Columns: snake_case
CREATE TABLE users (
    id              UUID PRIMARY KEY,
    email           VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP NOT NULL
);

-- Foreign keys: referenced_table_id
CREATE TABLE orders (
    id              UUID PRIMARY KEY,
    user_id         UUID REFERENCES users(id),
    shipping_address_id UUID REFERENCES addresses(id)
);

-- Junction tables: table1_table2 (alphabetical)
CREATE TABLE products_tags (
    product_id UUID REFERENCES products(id),
    tag_id     UUID REFERENCES tags(id)
);
```

### Environment Variables

```bash
# Pattern: SCREAMING_SNAKE_CASE
DATABASE_URL=postgres://localhost/mydb
API_SECRET_KEY=xxx
MAX_RETRY_COUNT=3
ENABLE_DEBUG_MODE=true

# Prefixes for grouping
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp

AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_REGION=us-east-1
```

---

## Language-Specific Conventions

### JavaScript/TypeScript

```typescript
// Variables and functions: camelCase
const userName = "John";
function getUserById(id: string) { }

// Classes and types: PascalCase
class UserService { }
interface UserRepository { }
type OrderStatus = string;

// Constants: SCREAMING_SNAKE_CASE or camelCase
const MAX_RETRY_COUNT = 3;
const apiBaseUrl = "https://api.example.com";

// Private members: _prefix or #
class User {
  private _password: string;
  #internalState: any;
}
```

### Python

```python
# Variables and functions: snake_case
user_name = "John"
def get_user_by_id(user_id: str):
    pass

# Classes: PascalCase
class UserService:
    pass

# Constants: SCREAMING_SNAKE_CASE
MAX_RETRY_COUNT = 3
API_BASE_URL = "https://api.example.com"

# Private members: _prefix
class User:
    def __init__(self):
        self._password = None      # Protected
        self.__internal = None     # Private (name mangling)
```

### Java

```java
// Variables and methods: camelCase
String userName = "John";
public User getUserById(String id) { }

// Classes and interfaces: PascalCase
public class UserService { }
public interface UserRepository { }

// Constants: SCREAMING_SNAKE_CASE
public static final int MAX_RETRY_COUNT = 3;

// Packages: lowercase, dot-separated
package com.example.myapp.services;
```

### Go

```go
// Exported (public): PascalCase
func GetUserById(id string) User { }
type UserService struct { }
const MaxRetryCount = 3

// Unexported (private): camelCase
func getUserFromCache(id string) user { }
type userCache struct { }
const defaultTimeout = 30

// Packages: lowercase, single word preferred
package user
package orderservice  // or order (shorter is better)
```

---

## Common Naming Mistakes

| Mistake | Example | Problem | Better |
|---------|---------|---------|--------|
| Too short | `u`, `o`, `p` | Unclear meaning | `user`, `order`, `product` |
| Too long | `userAccountInformationData` | Hard to read | `userAccount` |
| Misleading | `accountList` (when it's a Map) | Incorrect type hint | `accountsById` |
| Inconsistent | `getUser` / `fetchOrder` | Different words for same concept | Pick one: `get` or `fetch` |
| Redundant type | `userObject`, `nameString` | Unnecessary suffix | `user`, `name` |
| Negative boolean | `isNotActive`, `hasNoPermission` | Double negative confusion | `isInactive`, `lacksPermission` |
| Generic | `data`, `info`, `manager` | No specific meaning | `userData`, `orderInfo`, `userManager` |
| Implementation | `arrayOfUsers`, `hashMapOfPrices` | Exposes implementation | `users`, `prices` |

---

## Naming Checklist

### Before Committing

- [ ] Names are pronounceable
- [ ] Names reveal intent
- [ ] Consistent with existing codebase
- [ ] No cryptic abbreviations
- [ ] Booleans read as yes/no questions
- [ ] Collections are plural
- [ ] Functions describe action
- [ ] Classes describe thing/concept
- [ ] No misleading names
- [ ] Searchable (not too generic)
