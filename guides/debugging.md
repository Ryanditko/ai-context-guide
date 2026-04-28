# Debugging

Guide for systematic debugging techniques and strategies.

## Debugging Mindset

### The Scientific Method

```
1. Observe    → What exactly is happening?
2. Hypothesize → What might be causing it?
3. Predict    → If hypothesis is correct, what should I see?
4. Test       → Run experiment to verify
5. Iterate    → Refine hypothesis based on results
```

### Common Debugging Mistakes

| Mistake | Problem | Better Approach |
|---------|---------|-----------------|
| Changing code randomly | Wastes time, may introduce new bugs | Form hypothesis first |
| Assuming you know the cause | Confirmation bias | Verify with evidence |
| Not reading error messages | Missing obvious clues | Read every word carefully |
| Debugging in production | Risky, hard to reproduce | Reproduce locally first |
| Not taking breaks | Tunnel vision | Step away, fresh eyes help |

---

## Error Analysis

### Reading Stack Traces

```python
# Python stack trace
Traceback (most recent call last):
  File "app.py", line 45, in process_order    # ← Called from here
    total = calculate_total(items)
  File "pricing.py", line 23, in calculate_total  # ← Then here
    discount = apply_discount(subtotal, code)
  File "pricing.py", line 67, in apply_discount   # ← Error here
    rate = discount_rates[code]
KeyError: 'INVALID_CODE'                          # ← Actual error
```

**Reading order**: Bottom to top for the error, top to bottom for the call chain.

```javascript
// JavaScript stack trace
TypeError: Cannot read property 'name' of undefined
    at getUserName (user.js:15:24)        // ← Error location
    at processUser (handler.js:42:10)     // ← Called from
    at async handleRequest (server.js:88) // ← Origin
```

### Common Error Patterns

| Error Type | Typical Cause | First Check |
|------------|---------------|-------------|
| NullPointerException | Uninitialized variable | Where is the variable set? |
| KeyError / undefined | Missing key in dict/object | Log the actual keys |
| IndexError | Array bounds | Check array length |
| TypeError | Wrong type passed | Log `typeof` / `type()` |
| Connection refused | Service not running | Check service status |
| Timeout | Slow query/network | Check response times |

---

## Debugging Techniques

### 1. Binary Search (Bisecting)

When bug appeared between two known states:

```bash
# Git bisect
git bisect start
git bisect bad HEAD          # Current (broken)
git bisect good v1.2.0       # Last known good
# Git checks out middle commit
# Test and mark:
git bisect good  # or
git bisect bad
# Repeat until found
git bisect reset
```

For code, comment out half and test:

```python
def complex_function():
    step_1()
    step_2()
    # step_3()  # Comment out half
    # step_4()
    # If bug gone, it's in commented half
```

### 2. Rubber Duck Debugging

Explain the problem out loud:

```
"So the function receives user_id...
 then it queries the database...
 wait, I'm not checking if the result is None!"
```

### 3. Print/Log Debugging

```python
# Strategic logging
def process_order(order):
    logger.debug(f"Processing order: {order.id}")
    logger.debug(f"Items count: {len(order.items)}")
    
    for i, item in enumerate(order.items):
        logger.debug(f"Item {i}: {item.sku}, qty={item.quantity}")
        
    total = calculate_total(order.items)
    logger.debug(f"Calculated total: {total}")
    
    if total > order.limit:
        logger.warning(f"Order {order.id} exceeds limit: {total} > {order.limit}")
```

**Log what matters:**
- Function entry/exit with key parameters
- Loop iterations (first, last, or sampled)
- Decision points (if/else branches taken)
- Values before operations that might fail

### 4. Divide and Conquer

```python
# Isolate the problem
def debug_order_processing():
    # Step 1: Test data fetching
    order = get_order("test-123")
    print(f"Order loaded: {order is not None}")
    
    # Step 2: Test calculation
    items = [Item(price=10, qty=2), Item(price=5, qty=1)]
    total = calculate_total(items)
    print(f"Total calculated: {total}")  # Should be 25
    
    # Step 3: Test payment
    result = mock_payment(100)
    print(f"Payment result: {result}")
```

### 5. Minimal Reproduction

Create the smallest code that reproduces the bug:

```python
# Original: 500 lines of code with bug

# Minimal reproduction:
def test_bug():
    data = {"users": [{"id": 1}]}
    result = process(data)  # Crashes here
    
# Even simpler:
process({"users": [{"id": 1}]})  # Still crashes

# Simplest:
process({"users": []})  # Works
process({"users": [{}]})  # Crashes! Bug: missing 'id' check
```

---

## Debugger Usage

### Python (pdb/ipdb)

```python
# Insert breakpoint
import pdb; pdb.set_trace()  # Python < 3.7
breakpoint()                  # Python 3.7+

# Common commands
# n (next)     - Execute next line
# s (step)     - Step into function
# c (continue) - Continue to next breakpoint
# p expr       - Print expression
# pp expr      - Pretty print
# l (list)     - Show current code
# w (where)    - Show stack trace
# u (up)       - Go up in stack
# d (down)     - Go down in stack
# b line       - Set breakpoint at line
# cl           - Clear breakpoints

# Conditional breakpoint
pdb.set_trace() if user_id == "problem-user" else None

# Post-mortem debugging
python -m pdb -c continue script.py  # Break on exception
```

### JavaScript (Node.js)

```javascript
// Insert breakpoint
debugger;

// Run with inspector
// node --inspect script.js
// node --inspect-brk script.js  // Break at start

// Chrome DevTools: chrome://inspect

// VS Code launch.json
{
  "type": "node",
  "request": "launch",
  "name": "Debug",
  "program": "${workspaceFolder}/src/index.js",
  "skipFiles": ["<node_internals>/**"]
}
```

### VS Code Debugging

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Current File",
      "type": "python",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal",
      "justMyCode": false
    },
    {
      "name": "Python: FastAPI",
      "type": "python",
      "request": "launch",
      "module": "uvicorn",
      "args": ["main:app", "--reload"],
      "jinja": true
    },
    {
      "name": "Node: Current File",
      "type": "node",
      "request": "launch",
      "program": "${file}"
    },
    {
      "name": "Attach to Process",
      "type": "node",
      "request": "attach",
      "port": 9229
    }
  ]
}
```

---

## Debugging Specific Issues

### Memory Leaks

```python
# Python memory profiling
import tracemalloc

tracemalloc.start()

# ... run code ...

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

print("Top 10 memory allocations:")
for stat in top_stats[:10]:
    print(stat)
```

```javascript
// Node.js memory debugging
// Run with: node --inspect --expose-gc script.js

// In Chrome DevTools:
// 1. Take heap snapshot
// 2. Run operation
// 3. Take another snapshot
// 4. Compare snapshots

// Programmatic
const v8 = require('v8');
const heapStats = v8.getHeapStatistics();
console.log('Heap used:', heapStats.used_heap_size / 1024 / 1024, 'MB');
```

### Performance Issues

```python
# Python profiling
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

# ... code to profile ...

profiler.disable()
stats = pstats.Stats(profiler).sort_stats('cumulative')
stats.print_stats(20)  # Top 20 functions

# Line profiler (pip install line_profiler)
@profile
def slow_function():
    # ...
# Run: kernprof -l -v script.py
```

```javascript
// Node.js profiling
console.time('operation');
// ... code ...
console.timeEnd('operation');

// CPU profiling
const { performance, PerformanceObserver } = require('perf_hooks');

const obs = new PerformanceObserver((items) => {
  console.log(items.getEntries());
});
obs.observe({ entryTypes: ['measure'] });

performance.mark('start');
// ... code ...
performance.mark('end');
performance.measure('My Operation', 'start', 'end');
```

### Race Conditions

```python
# Add logging with timestamps
import logging
import threading

logging.basicConfig(
    format='%(asctime)s.%(msecs)03d [%(threadName)s] %(message)s',
    datefmt='%H:%M:%S',
    level=logging.DEBUG
)

def worker(name):
    logging.debug(f"{name} starting")
    # ... work ...
    logging.debug(f"{name} acquiring lock")
    with lock:
        logging.debug(f"{name} has lock")
        # ... critical section ...
    logging.debug(f"{name} released lock")
```

### Network Issues

```python
# Verbose HTTP debugging
import http.client
import logging

http.client.HTTPConnection.debuglevel = 1
logging.basicConfig()
logging.getLogger().setLevel(logging.DEBUG)
requests_log = logging.getLogger("urllib3")
requests_log.setLevel(logging.DEBUG)
requests_log.propagate = True

# Now all HTTP traffic is logged
import requests
requests.get("https://api.example.com")
```

```bash
# Network debugging tools
curl -v https://api.example.com        # Verbose output
curl -w "@curl-format.txt" URL         # Timing breakdown

# curl-format.txt:
#     time_namelookup:  %{time_namelookup}\n
#        time_connect:  %{time_connect}\n
#     time_appconnect:  %{time_appconnect}\n
#    time_pretransfer:  %{time_pretransfer}\n
#       time_redirect:  %{time_redirect}\n
#  time_starttransfer:  %{time_starttransfer}\n
#                     ----------\n
#          time_total:  %{time_total}\n

# DNS debugging
dig example.com
nslookup example.com

# TCP debugging
nc -zv host port
telnet host port
```

### Database Issues

```sql
-- Query analysis
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- Check locks (PostgreSQL)
SELECT * FROM pg_locks WHERE NOT granted;

-- Active queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Kill long-running query
SELECT pg_cancel_backend(pid);
```

```python
# SQLAlchemy query logging
import logging
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)

# See actual SQL
from sqlalchemy import event

@event.listens_for(Engine, "before_cursor_execute")
def log_query(conn, cursor, statement, parameters, context, executemany):
    print(f"Query: {statement}")
    print(f"Parameters: {parameters}")
```

---

## Debugging in Production

### Safe Production Debugging

```python
# Feature flag for debug mode
if feature_flags.is_enabled("debug_mode", user_id=current_user.id):
    logger.setLevel(logging.DEBUG)

# Sampling
import random
if random.random() < 0.01:  # 1% of requests
    log_detailed_trace(request)

# Debug endpoints (protected!)
@app.get("/debug/config")
@require_admin
def debug_config():
    return {
        "environment": os.environ.get("ENV"),
        "feature_flags": get_all_flags(),
        "version": get_version(),
    }
```

### Distributed Tracing

```python
# OpenTelemetry setup
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# Usage
with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order.id", order_id)
    # ... processing ...
    span.add_event("payment_processed")
```

### Log Analysis

```bash
# Search logs
grep -r "ERROR" /var/log/app/
grep -A 5 -B 5 "order-12345" /var/log/app/  # Context around match

# Count errors by type
grep "ERROR" app.log | cut -d':' -f4 | sort | uniq -c | sort -rn

# Watch logs in real-time
tail -f /var/log/app/app.log | grep --line-buffered "ERROR"

# JSON logs with jq
cat app.log | jq 'select(.level == "error")'
cat app.log | jq 'select(.user_id == "123")'
```

---

## Debugging Tools

### Command Line

| Tool | Purpose | Example |
|------|---------|---------|
| `strace` | System calls | `strace -p PID` |
| `ltrace` | Library calls | `ltrace ./program` |
| `lsof` | Open files | `lsof -p PID` |
| `netstat` | Network connections | `netstat -tlnp` |
| `ss` | Socket stats | `ss -tlnp` |
| `tcpdump` | Network packets | `tcpdump -i eth0 port 80` |
| `htop` | Process monitor | `htop` |
| `dstat` | System stats | `dstat -cdngy` |

### IDE Features

| Feature | VS Code | JetBrains |
|---------|---------|-----------|
| Breakpoints | Click line number | Click gutter |
| Conditional | Right-click breakpoint | Right-click |
| Logpoints | Right-click → Logpoint | Evaluate and log |
| Watch | Add to Watch panel | Add to Watches |
| Call stack | Debug panel | Frames panel |
| Variable inspection | Hover or Variables panel | Hover or Variables |

### Browser DevTools

```javascript
// Console debugging
console.log('Simple log');
console.table([{a: 1}, {a: 2}]);  // Table format
console.group('Group');
console.log('Grouped item');
console.groupEnd();
console.time('timer');
// ... code ...
console.timeEnd('timer');
console.trace();  // Stack trace

// DOM debugging
$0  // Currently selected element
$_  // Last evaluated expression
$$('selector')  // querySelectorAll shorthand

// Network debugging
// Network tab → Filter by XHR → Right-click → Copy as cURL
```

---

## Debugging Checklist

### Before Starting
- [ ] Can you reproduce the bug?
- [ ] What changed recently?
- [ ] Read the full error message
- [ ] Check the logs

### During Investigation
- [ ] Form a hypothesis
- [ ] Test one thing at a time
- [ ] Keep notes of what you tried
- [ ] Simplify the reproduction case

### When Stuck
- [ ] Take a break
- [ ] Explain to someone (or rubber duck)
- [ ] Question your assumptions
- [ ] Look at related code
- [ ] Search for similar issues

### After Fixing
- [ ] Understand why it worked
- [ ] Add test to prevent regression
- [ ] Consider if similar bugs exist elsewhere
- [ ] Document the fix if non-obvious
