# Database Design

Guide for designing efficient, scalable, and maintainable database schemas.

## Design Principles

### 1. Understand the Domain First

Before designing tables, understand:
- What entities exist in the business domain?
- How do entities relate to each other?
- What queries will be most common?
- What are the data access patterns?

### 2. Normalization vs Denormalization

| Normal Form | Benefit | Trade-off |
|-------------|---------|-----------|
| 1NF | Atomic values | Basic structure |
| 2NF | No partial dependencies | Reduced redundancy |
| 3NF | No transitive dependencies | More joins needed |
| BCNF | Stricter 3NF | Even more joins |

**Rule of thumb**: Normalize first, denormalize for specific performance needs.

---

## Schema Design

### Naming Conventions

```sql
-- Tables: plural, snake_case
CREATE TABLE users (...);
CREATE TABLE order_items (...);
CREATE TABLE user_preferences (...);

-- Columns: snake_case, descriptive
CREATE TABLE orders (
    id              UUID PRIMARY KEY,
    user_id         UUID NOT NULL,        -- Foreign key
    total_amount    DECIMAL(10, 2),       -- With unit hint
    currency_code   CHAR(3),              -- ISO standard
    status          VARCHAR(20),
    is_paid         BOOLEAN DEFAULT FALSE, -- Boolean prefix
    created_at      TIMESTAMP NOT NULL,    -- Timestamp suffix
    updated_at      TIMESTAMP NOT NULL,
    deleted_at      TIMESTAMP              -- Soft delete
);

-- Indexes: idx_table_columns
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status_created ON orders(status, created_at);

-- Foreign keys: fk_table_referenced
ALTER TABLE orders
ADD CONSTRAINT fk_orders_users
FOREIGN KEY (user_id) REFERENCES users(id);
```

### Primary Keys

```sql
-- Option 1: UUID (distributed systems)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid()
);

-- Option 2: Serial/Identity (simpler, sequential)
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);

-- Option 3: ULID (sortable UUID)
-- Use application-generated ULIDs for time-ordered UUIDs
```

| Type | Pros | Cons |
|------|------|------|
| UUID | Distributed generation, no collisions | Larger (16 bytes), random order |
| Serial | Small, sequential, readable | Central generation, predictable |
| ULID | Sortable, distributed | Less common, 128-bit |

### Relationships

```sql
-- One-to-Many
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id)
);

-- Many-to-Many (junction table)
CREATE TABLE product_categories (
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, category_id)
);

-- One-to-One
CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_url VARCHAR(500)
);

-- Self-referencing (hierarchies)
CREATE TABLE categories (
    id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id UUID REFERENCES categories(id)
);
```

### Soft Deletes

```sql
-- Soft delete pattern
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    deleted_at TIMESTAMP,
    
    -- Unique constraint only for non-deleted
    CONSTRAINT unique_email_when_active 
        UNIQUE NULLS NOT DISTINCT (email, deleted_at)
);

-- Query active records only
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;

-- Alternative: status column
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    status VARCHAR(20) NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'archived', 'deleted'))
);
```

### Audit Columns

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    -- ... business columns ...
    
    -- Audit columns
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by UUID REFERENCES users(id),
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID REFERENCES users(id),
    version INTEGER NOT NULL DEFAULT 1  -- Optimistic locking
);

-- Auto-update trigger
CREATE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    NEW.version = OLD.version + 1;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

---

## Data Types

### Choosing the Right Type

| Data | Type | Notes |
|------|------|-------|
| ID | UUID / BIGINT | UUID for distributed |
| Money | DECIMAL(19, 4) | Never use FLOAT |
| Percentages | DECIMAL(5, 2) | Allows 100.00% |
| Email | VARCHAR(254) | Max email length |
| URL | VARCHAR(2048) | Max URL length |
| Phone | VARCHAR(20) | Include country code |
| IP Address | INET | PostgreSQL specific |
| JSON | JSONB | PostgreSQL, indexed |
| Timestamps | TIMESTAMPTZ | Always with timezone |
| Booleans | BOOLEAN | Not nullable if possible |
| Enums | VARCHAR + CHECK | Or native ENUM |

### Enum Handling

```sql
-- Option 1: Native ENUM (PostgreSQL)
CREATE TYPE order_status AS ENUM ('pending', 'confirmed', 'shipped', 'delivered');

CREATE TABLE orders (
    status order_status NOT NULL DEFAULT 'pending'
);

-- Option 2: VARCHAR with CHECK (more portable)
CREATE TABLE orders (
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered'))
);

-- Option 3: Reference table (most flexible)
CREATE TABLE order_statuses (
    code VARCHAR(20) PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    sort_order INTEGER
);

INSERT INTO order_statuses VALUES 
    ('pending', 'Pending', 1),
    ('confirmed', 'Confirmed', 2),
    ('shipped', 'Shipped', 3),
    ('delivered', 'Delivered', 4);

CREATE TABLE orders (
    status VARCHAR(20) NOT NULL REFERENCES order_statuses(code)
);
```

### JSON Columns

```sql
-- Store flexible data
CREATE TABLE products (
    id UUID PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    attributes JSONB NOT NULL DEFAULT '{}'
);

-- Query JSON
SELECT * FROM products
WHERE attributes->>'color' = 'red';

SELECT * FROM products
WHERE attributes @> '{"size": "large"}';

-- Index JSON paths
CREATE INDEX idx_products_color 
ON products ((attributes->>'color'));

CREATE INDEX idx_products_attrs 
ON products USING GIN (attributes);
```

---

## Indexing

### Index Types

```sql
-- B-Tree (default, most common)
CREATE INDEX idx_users_email ON users(email);

-- Partial index (subset of rows)
CREATE INDEX idx_orders_pending 
ON orders(created_at) 
WHERE status = 'pending';

-- Composite index
CREATE INDEX idx_orders_user_status 
ON orders(user_id, status, created_at DESC);

-- Unique index
CREATE UNIQUE INDEX idx_users_email_unique 
ON users(email) 
WHERE deleted_at IS NULL;

-- Covering index (include non-key columns)
CREATE INDEX idx_orders_user_covering 
ON orders(user_id) 
INCLUDE (status, total_amount);

-- Hash index (equality only)
CREATE INDEX idx_sessions_token 
ON sessions USING HASH (token);

-- GIN index (arrays, JSON, full-text)
CREATE INDEX idx_products_tags 
ON products USING GIN (tags);

-- GiST index (geometry, full-text)
CREATE INDEX idx_locations_coords 
ON locations USING GIST (coordinates);
```

### Index Strategy

```sql
-- Analyze query patterns
EXPLAIN ANALYZE 
SELECT * FROM orders 
WHERE user_id = '123' AND status = 'pending'
ORDER BY created_at DESC
LIMIT 10;

-- Create index for the query
CREATE INDEX idx_orders_user_status_created 
ON orders(user_id, status, created_at DESC);

-- Column order matters!
-- Put equality conditions first, then range/sort
-- Good: (user_id, status, created_at)
-- Bad:  (created_at, user_id, status)
```

### When to Index

| Index When | Don't Index When |
|------------|------------------|
| Foreign keys | Low selectivity columns |
| WHERE conditions | Rarely queried columns |
| JOIN columns | Small tables (< 1000 rows) |
| ORDER BY columns | Frequently updated columns |
| Unique constraints | LIKE '%prefix' queries |

---

## Query Optimization

### EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
  AND o.created_at > '2024-01-01'
ORDER BY o.created_at DESC
LIMIT 10;

-- Read the output:
-- Seq Scan     → Consider adding index
-- Index Scan   → Good, using index
-- Bitmap Scan  → OK for moderate selectivity
-- Hash Join    → Good for equality joins
-- Nested Loop  → Check if table is small or indexed
-- Sort         → Consider index with ORDER BY columns
```

### Common Optimizations

```sql
-- ❌ Bad: SELECT *
SELECT * FROM orders WHERE user_id = '123';

-- ✅ Good: Select only needed columns
SELECT id, status, total_amount 
FROM orders 
WHERE user_id = '123';

-- ❌ Bad: N+1 queries
FOR each user:
    SELECT * FROM orders WHERE user_id = user.id;

-- ✅ Good: Single query with JOIN or IN
SELECT * FROM orders 
WHERE user_id IN (SELECT id FROM users WHERE status = 'active');

-- ❌ Bad: Function on indexed column
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- ✅ Good: Store normalized or use expression index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- ❌ Bad: OR on different columns
SELECT * FROM orders WHERE user_id = '1' OR status = 'pending';

-- ✅ Good: UNION (uses indexes on both)
SELECT * FROM orders WHERE user_id = '1'
UNION
SELECT * FROM orders WHERE status = 'pending';

-- ❌ Bad: Unbounded queries
SELECT * FROM orders WHERE status = 'pending';

-- ✅ Good: Always paginate
SELECT * FROM orders 
WHERE status = 'pending' 
ORDER BY created_at DESC 
LIMIT 50 OFFSET 0;
```

### Pagination

```sql
-- Offset pagination (simple but slow for large offsets)
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 100;  -- Slow for OFFSET 100000

-- Cursor pagination (better performance)
SELECT * FROM orders
WHERE created_at < '2024-01-15T10:00:00Z'  -- Last item from previous page
ORDER BY created_at DESC
LIMIT 20;

-- Keyset pagination with composite key
SELECT * FROM orders
WHERE (created_at, id) < ('2024-01-15T10:00:00Z', 'abc-123')
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

---

## Migrations

### Migration Best Practices

```sql
-- migrations/001_create_users.sql
-- Always include both up and down migrations

-- Up
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);

-- Down
DROP TABLE users;
```

### Safe Migration Patterns

```sql
-- ✅ Safe: Add nullable column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- ✅ Safe: Add column with default (PostgreSQL 11+)
ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'active';

-- ⚠️ Careful: Add NOT NULL requires default or backfill
-- Step 1: Add nullable
ALTER TABLE users ADD COLUMN verified BOOLEAN;
-- Step 2: Backfill
UPDATE users SET verified = FALSE WHERE verified IS NULL;
-- Step 3: Add constraint
ALTER TABLE users ALTER COLUMN verified SET NOT NULL;
ALTER TABLE users ALTER COLUMN verified SET DEFAULT FALSE;

-- ✅ Safe: Create index concurrently
CREATE INDEX CONCURRENTLY idx_users_name ON users(name);

-- ⚠️ Careful: Rename column (may break app)
-- Use view or alias during transition
ALTER TABLE users RENAME COLUMN name TO full_name;

-- ✅ Safe: Add foreign key (validate separately)
ALTER TABLE orders 
ADD CONSTRAINT fk_orders_users 
FOREIGN KEY (user_id) REFERENCES users(id) 
NOT VALID;

-- Later, validate without locking
ALTER TABLE orders 
VALIDATE CONSTRAINT fk_orders_users;
```

### Migration Checklist

- [ ] Forward and backward compatible
- [ ] Tested on copy of production data
- [ ] Index creation is CONCURRENT
- [ ] Long-running operations estimated
- [ ] Rollback plan prepared
- [ ] Application handles both old and new schema

---

## Common Patterns

### Event Sourcing

```sql
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(50) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version INTEGER NOT NULL
);

CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, version);
CREATE INDEX idx_events_type ON events(event_type, created_at);

-- Ensure version ordering
CREATE UNIQUE INDEX idx_events_aggregate_version 
ON events(aggregate_type, aggregate_id, version);
```

### Multi-Tenancy

```sql
-- Option 1: Shared table with tenant_id
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    -- ... other columns ...
    
    CONSTRAINT unique_order_per_tenant UNIQUE (tenant_id, id)
);

CREATE INDEX idx_orders_tenant ON orders(tenant_id);

-- Row-level security
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::UUID);

-- Option 2: Schema per tenant
CREATE SCHEMA tenant_abc;
CREATE TABLE tenant_abc.orders (...);
```

### Hierarchical Data

```sql
-- Adjacency list (simple, recursive queries)
CREATE TABLE categories (
    id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id UUID REFERENCES categories(id)
);

-- Recursive query
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS depth
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;

-- Materialized path (fast reads)
CREATE TABLE categories (
    id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    path VARCHAR(500) NOT NULL  -- '/root/parent/child'
);

CREATE INDEX idx_categories_path ON categories(path);

-- Query descendants
SELECT * FROM categories WHERE path LIKE '/electronics/%';
```

### Temporal Data

```sql
-- Valid time (business time)
CREATE TABLE product_prices (
    id UUID PRIMARY KEY,
    product_id UUID NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    valid_from TIMESTAMP NOT NULL,
    valid_to TIMESTAMP,
    
    -- Prevent overlapping periods
    EXCLUDE USING GIST (
        product_id WITH =,
        tstzrange(valid_from, valid_to, '[)') WITH &&
    )
);

-- Get current price
SELECT * FROM product_prices
WHERE product_id = '123'
  AND valid_from <= CURRENT_TIMESTAMP
  AND (valid_to IS NULL OR valid_to > CURRENT_TIMESTAMP);

-- Get price at specific time
SELECT * FROM product_prices
WHERE product_id = '123'
  AND valid_from <= '2024-01-15'
  AND (valid_to IS NULL OR valid_to > '2024-01-15');
```

---

## Performance Monitoring

### Key Metrics

```sql
-- Table size
SELECT 
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS data_size,
    pg_size_pretty(pg_indexes_size(relid)) AS index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Index usage
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS times_used,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Slow queries
SELECT
    query,
    calls,
    total_time / 1000 AS total_seconds,
    mean_time / 1000 AS mean_seconds,
    rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;

-- Cache hit ratio (should be > 99%)
SELECT
    sum(heap_blks_read) AS heap_read,
    sum(heap_blks_hit) AS heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS ratio
FROM pg_statio_user_tables;
```

---

## Database Design Checklist

### Schema Design
- [ ] Tables are properly normalized
- [ ] Primary keys are defined
- [ ] Foreign keys with appropriate actions
- [ ] Appropriate data types
- [ ] Constraints for data integrity
- [ ] Audit columns where needed

### Performance
- [ ] Indexes on foreign keys
- [ ] Indexes on WHERE clause columns
- [ ] Indexes on ORDER BY columns
- [ ] No over-indexing
- [ ] Query plans analyzed

### Operations
- [ ] Migrations are reversible
- [ ] Backup strategy defined
- [ ] Monitoring in place
- [ ] Connection pooling configured
- [ ] Query timeout configured
