# Performance

Guide for performance optimization in applications.

## Fundamental Principles

### 1. Measure First
Never optimize without data.

```python
# Use profilers and metrics
import cProfile
import pstats

def profile_function(func):
    profiler = cProfile.Profile()
    profiler.enable()
    result = func()
    profiler.disable()
    stats = pstats.Stats(profiler).sort_stats('cumulative')
    stats.print_stats(10)  # Top 10 slowest functions
    return result
```

### 2. Optimize the Right Thing
Focus on real bottlenecks (Amdahl's Law).

```
If 90% of time is in one function:
- Optimizing it by 50% = 45% total improvement
- Optimizing another by 50% = 5% total improvement
```

### 3. Premature Optimization
"Premature optimization is the root of all evil" - Knuth

```
1. Make it work (correct)
2. Make it right (clean)
3. Make it fast (optimized) - only if necessary
```

## Database Performance

### N+1 Queries

```python
# ❌ Bad: N+1 queries
users = User.query.all()
for user in users:
    orders = Order.query.filter_by(user_id=user.id).all()  # N queries

# ✅ Good: Eager loading
users = User.query.options(joinedload(User.orders)).all()  # 1 query

# ✅ Good: Batch loading
user_ids = [u.id for u in users]
orders = Order.query.filter(Order.user_id.in_(user_ids)).all()  # 2 queries
```

### Indexing

```sql
-- Identify slow queries
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

-- Indexes for frequent queries
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status_created ON orders(status, created_at);

-- Composite index for queries with multiple conditions
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### Query Optimization

```python
# ❌ Bad: Loads everything to count
count = len(Order.query.filter_by(status='pending').all())

# ✅ Good: Count in database
count = Order.query.filter_by(status='pending').count()

# ❌ Bad: SELECT * when only some fields needed
users = User.query.all()
emails = [u.email for u in users]

# ✅ Good: SELECT only necessary fields
emails = db.session.query(User.email).all()
```

## Caching

### Cache Layers

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│    CDN      │────▶│   Server    │
│   Cache     │     │   Cache     │     │   Cache     │
│ (browser)   │     │ (edge)      │     │ (Redis)     │
└─────────────┘     └─────────────┘     └─────────────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │  Database   │
                                        └─────────────┘
```

### Application Cache

```python
from functools import lru_cache
import redis

# In-memory cache for immutable data
@lru_cache(maxsize=1000)
def get_config(key: str) -> str:
    return db.query(Config).filter_by(key=key).first().value

# Redis for distributed cache
redis_client = redis.Redis()

def get_user_profile(user_id: str) -> dict:
    cache_key = f"user:profile:{user_id}"
    
    # Try cache first
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Fetch from DB
    profile = db.query(User).get(user_id).to_dict()
    
    # Cache with TTL
    redis_client.setex(cache_key, 3600, json.dumps(profile))
    
    return profile

def invalidate_user_cache(user_id: str):
    """Invalidate cache when data changes."""
    redis_client.delete(f"user:profile:{user_id}")
```

### Cache Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Cache-Aside | App manages cache | Frequently read data |
| Write-Through | Writes to cache and DB | Consistency important |
| Write-Behind | Writes to cache, async to DB | High write performance |
| Read-Through | Cache fetches from DB | Simplifies code |

## Async and Concurrency

### I/O Bound Operations

```python
import asyncio
import aiohttp

# ❌ Bad: Sequential
def fetch_all_sync(urls: list[str]):
    results = []
    for url in urls:
        response = requests.get(url)
        results.append(response.json())
    return results  # N * latency

# ✅ Good: Concurrent
async def fetch_all_async(urls: list[str]):
    async with aiohttp.ClientSession() as session:
        tasks = [session.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return [await r.json() for r in responses]  # max(latencies)
```

### Connection Pooling

```python
# Database connection pool
from sqlalchemy import create_engine

engine = create_engine(
    DATABASE_URL,
    pool_size=20,        # Connections kept
    max_overflow=10,     # Extra connections if needed
    pool_timeout=30,     # Timeout to get connection
    pool_recycle=1800,   # Recycle connections every 30min
)

# HTTP connection pool
import httpx

client = httpx.Client(
    limits=httpx.Limits(
        max_keepalive_connections=20,
        max_connections=100
    )
)
```

## Memory Optimization

### Generators vs Lists

```python
# ❌ Bad: Loads everything in memory
def process_large_file(path: str):
    lines = open(path).readlines()  # Loads entire file
    for line in lines:
        process(line)

# ✅ Good: Process line by line
def process_large_file(path: str):
    with open(path) as f:
        for line in f:  # Iterator, doesn't load everything
            process(line)

# ✅ Better: Generator expression
def get_valid_items(items):
    return (item for item in items if item.is_valid)  # Lazy evaluation
```

### Object Slots

```python
# ❌ Bad: Dict per instance (more memory)
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

# ✅ Good: Slots (less memory for many instances)
class Point:
    __slots__ = ['x', 'y']
    
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

## API Performance

### Pagination

```python
# Cursor-based pagination (better for large datasets)
@app.get("/orders")
def list_orders(cursor: str = None, limit: int = 20):
    query = Order.query.order_by(Order.id)
    
    if cursor:
        query = query.filter(Order.id > cursor)
    
    orders = query.limit(limit + 1).all()
    
    has_more = len(orders) > limit
    if has_more:
        orders = orders[:-1]
    
    return {
        "data": orders,
        "next_cursor": orders[-1].id if has_more else None
    }
```

### Response Compression

```python
from fastapi import FastAPI
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

### Selective Fields

```python
@app.get("/users/{user_id}")
def get_user(user_id: str, fields: str = None):
    user = User.query.get(user_id)
    
    if fields:
        requested = set(fields.split(','))
        return {k: v for k, v in user.to_dict().items() if k in requested}
    
    return user.to_dict()

# GET /users/123?fields=id,name,email
```

## Performance Monitoring

### Key Metrics

```python
from prometheus_client import Histogram, Counter

request_latency = Histogram(
    'http_request_duration_seconds',
    'Request latency',
    ['method', 'endpoint'],
    buckets=[.01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

db_query_duration = Histogram(
    'db_query_duration_seconds',
    'Database query latency',
    ['operation', 'table']
)

cache_hits = Counter(
    'cache_hits_total',
    'Cache hit count',
    ['cache_name']
)

cache_misses = Counter(
    'cache_misses_total',
    'Cache miss count',
    ['cache_name']
)
```

### Typical SLOs

| Metric | Target |
|--------|--------|
| P50 latency | < 100ms |
| P95 latency | < 500ms |
| P99 latency | < 1s |
| Error rate | < 0.1% |
| Availability | > 99.9% |

## Performance Checklist

### Database
- [ ] Queries analyzed with EXPLAIN
- [ ] Indexes for frequent queries
- [ ] N+1 queries eliminated
- [ ] Connection pooling configured

### Caching
- [ ] Cache on frequent queries
- [ ] Appropriate TTL
- [ ] Correct invalidation
- [ ] Cache hit rate monitored

### API
- [ ] Pagination implemented
- [ ] Compression enabled
- [ ] Timeouts configured
- [ ] Rate limiting active

### Code
- [ ] Profiling performed
- [ ] Hotspots identified
- [ ] Memory leaks verified
- [ ] Async where appropriate
