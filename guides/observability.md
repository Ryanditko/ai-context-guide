# Observability

Guide for logging, metrics, and tracing in distributed systems.

## The Three Pillars

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│    LOGS     │  │   METRICS   │  │   TRACES    │
│             │  │             │  │             │
│ What        │  │ How much    │  │ Where       │
│ happened    │  │ and when    │  │ and why     │
└─────────────┘  └─────────────┘  └─────────────┘
```

## Logging

### Log Levels

| Level | Usage |
|-------|-------|
| `DEBUG` | Detailed information for debugging |
| `INFO` | Normal system events |
| `WARN` | Unexpected but recoverable situation |
| `ERROR` | Error that affects an operation |
| `FATAL` | Critical error that terminates the system |

### Structured Logging

```python
# Bad: Unstructured log
logger.info(f"User {user_id} placed order {order_id} for ${amount}")

# Good: Structured log (JSON)
logger.info(
    "Order placed",
    extra={
        "user_id": user_id,
        "order_id": order_id,
        "amount": amount,
        "currency": "USD",
        "items_count": len(items)
    }
)

# Output:
# {"level": "INFO", "message": "Order placed", "user_id": "123", 
#  "order_id": "456", "amount": 99.99, "timestamp": "2024-01-15T10:30:00Z"}
```

### Request Context

```python
# Propagate context through the request
import contextvars

request_id = contextvars.ContextVar('request_id')
user_id = contextvars.ContextVar('user_id')

class ContextFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id.get(None)
        record.user_id = user_id.get(None)
        return True
```

### What to log

```python
# ✅ Good
logger.info("Payment processed", extra={"order_id": "123", "status": "success"})
logger.error("External API failed", extra={"service": "payment-gateway", "status_code": 503})
logger.warn("Rate limit approaching", extra={"current": 95, "limit": 100})

# ❌ Avoid
logger.debug(f"Entering function with params: {params}")  # Too verbose
logger.info(f"Password: {password}")  # NEVER log sensitive data
logger.error("Error occurred")  # No useful context
```

### PII and Sensitive Data

```python
SENSITIVE_FIELDS = {"password", "ssn", "card_number", "cvv", "token"}

def sanitize(data: dict) -> dict:
    """Remove or mask sensitive data before logging."""
    return {
        k: "***REDACTED***" if k in SENSITIVE_FIELDS else v
        for k, v in data.items()
    }

logger.info("User updated", extra=sanitize(user_data))
```

## Metrics

### Golden Signals

```
┌────────────────────────────────────────────────────────┐
│                    GOLDEN SIGNALS                       │
├──────────────┬─────────────────────────────────────────┤
│ Latency      │ Request response time                   │
│ Traffic      │ Requests per second                     │
│ Errors       │ Rate of failing requests                │
│ Saturation   │ How "full" the system is                │
└──────────────┴─────────────────────────────────────────┘
```

### Metric Types

```python
from prometheus_client import Counter, Histogram, Gauge

# Counter: Always increments
requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)
requests_total.labels(method='GET', endpoint='/users', status='200').inc()

# Histogram: Distribution of values
request_latency = Histogram(
    'http_request_duration_seconds',
    'Request latency in seconds',
    ['endpoint'],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)
with request_latency.labels(endpoint='/users').time():
    handle_request()

# Gauge: Value that can go up or down
active_connections = Gauge(
    'active_connections',
    'Number of active connections'
)
active_connections.inc()
active_connections.dec()
```

### Naming Convention

```
# Pattern: <namespace>_<name>_<unit>

# Good
http_requests_total
http_request_duration_seconds
db_connections_active
queue_messages_pending

# Bad
requests          # Too generic
latency           # No unit
numErrors         # Doesn't follow pattern
```

### Alerts

```yaml
# Example Prometheus AlertManager
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "P95 latency above 2 seconds"
```

## Distributed Tracing

### Concepts

```
Trace: Complete journey of a request through the system
  └── Span: An individual operation within the trace
        ├── Span ID
        ├── Parent Span ID
        ├── Operation Name
        ├── Start/End Time
        ├── Tags (metadata)
        └── Logs (events)
```

### Implementation with OpenTelemetry

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider

tracer = trace.get_tracer(__name__)

@tracer.start_as_current_span("process_order")
def process_order(order_id: str):
    span = trace.get_current_span()
    span.set_attribute("order.id", order_id)
    
    with tracer.start_as_current_span("validate_order"):
        validate(order)
    
    with tracer.start_as_current_span("charge_payment"):
        charge(order)
    
    with tracer.start_as_current_span("send_notification"):
        notify(order)
```

### Context Propagation

```python
# HTTP Client - propagates trace context
import requests
from opentelemetry.propagate import inject

def call_external_service(url: str):
    headers = {}
    inject(headers)  # Injects trace context into headers
    return requests.get(url, headers=headers)

# HTTP Server - extracts trace context
from opentelemetry.propagate import extract

def handle_request(request):
    context = extract(request.headers)
    with tracer.start_as_current_span("handle_request", context=context):
        ...
```

## Dashboards

### Recommended Layout

```
┌─────────────────────────────────────────────────────────┐
│                     OVERVIEW                             │
│  [Request Rate] [Error Rate] [P95 Latency] [Saturation] │
├─────────────────────────────────────────────────────────┤
│                     TRAFFIC                              │
│  [Requests by endpoint] [Requests by status code]       │
├─────────────────────────────────────────────────────────┤
│                    LATENCY                               │
│  [P50/P95/P99 over time] [Latency by endpoint]         │
├─────────────────────────────────────────────────────────┤
│                     ERRORS                               │
│  [Error rate over time] [Top errors] [Recent errors]   │
├─────────────────────────────────────────────────────────┤
│                  DEPENDENCIES                            │
│  [Database latency] [External APIs] [Cache hit rate]   │
└─────────────────────────────────────────────────────────┘
```

## Observability Checklist

### Logging
- [ ] Structured logs (JSON)
- [ ] Request ID in all logs
- [ ] Appropriate log levels
- [ ] PII/sensitive data redacted
- [ ] Retention configured

### Metrics
- [ ] Golden signals implemented
- [ ] Dependency metrics
- [ ] Alerts configured
- [ ] Dashboards created

### Tracing
- [ ] Traces on critical operations
- [ ] Context propagation between services
- [ ] Adequate sampling rate
- [ ] Integration with tools (Jaeger, Zipkin)
