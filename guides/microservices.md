# Microservices

Guide for designing, building, and operating microservices architectures.

## When to Use Microservices

### Good Fit

- Large teams that need to work independently
- Different parts of system need different scaling
- Different technology requirements per component
- Need for independent deployability
- Complex domain with clear bounded contexts

### Not a Good Fit

- Small team (< 5 developers)
- Simple application
- Unclear domain boundaries
- Need for strong consistency across data
- Limited DevOps capability

### Monolith First

> "Don't start with microservices. Start with a well-structured monolith, and extract services when you have clear bounded contexts and team scaling needs."

---

## Service Design

### Bounded Contexts

```
┌─────────────────────────────────────────────────────────────────┐
│                         E-Commerce Domain                        │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Order Context  │ Inventory Ctx   │     Customer Context        │
│  ┌───────────┐  │  ┌───────────┐  │  ┌───────────────────────┐  │
│  │   Order   │  │  │  Product  │  │  │       Customer        │  │
│  │   Item    │  │  │   Stock   │  │  │       Address         │  │
│  │  Payment  │  │  │ Warehouse │  │  │    Preferences        │  │
│  └───────────┘  │  └───────────┘  │  └───────────────────────┘  │
│                 │                 │                             │
│  Order Service  │ Inventory Svc   │     Customer Service        │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

### Service Boundaries

| Good Boundary | Bad Boundary |
|---------------|--------------|
| Business capability | Technical layer |
| Team ownership | Shared database |
| Independent lifecycle | Circular dependencies |
| Cohesive domain | Chatty interface |

### Service Size

**Too Large:**
- Multiple teams working on same service
- Long deployment cycles
- Many reasons to change

**Too Small:**
- Service can't function alone
- Every request needs multiple services
- Excessive network overhead

**Right Size:**
- Single team can own it
- Clear, cohesive responsibility
- Independent deployment

---

## Communication Patterns

### Synchronous (Request-Response)

```
┌──────────┐         ┌──────────┐         ┌──────────┐
│  Client  │  HTTP   │  Order   │  HTTP   │ Payment  │
│          │────────▶│ Service  │────────▶│ Service  │
│          │◀────────│          │◀────────│          │
└──────────┘         └──────────┘         └──────────┘
```

```python
# REST API call
class OrderService:
    def __init__(self, payment_client: PaymentClient):
        self.payment_client = payment_client
    
    def create_order(self, order: Order) -> OrderResult:
        # Validate order
        self._validate(order)
        
        # Save order (pending)
        saved_order = self.repository.save(order)
        
        # Call payment service
        try:
            payment_result = self.payment_client.charge(
                amount=order.total,
                customer_id=order.customer_id,
                idempotency_key=str(order.id)
            )
            
            if payment_result.success:
                saved_order.status = OrderStatus.CONFIRMED
            else:
                saved_order.status = OrderStatus.PAYMENT_FAILED
                
        except PaymentServiceError as e:
            saved_order.status = OrderStatus.PAYMENT_ERROR
            logger.error(f"Payment failed: {e}")
        
        self.repository.save(saved_order)
        return OrderResult(order=saved_order)
```

### Asynchronous (Event-Driven)

```
┌──────────┐         ┌─────────────┐         ┌──────────┐
│  Order   │  Event  │   Message   │  Event  │ Shipping │
│ Service  │────────▶│    Broker   │────────▶│ Service  │
│          │         │   (Kafka)   │         │          │
└──────────┘         └─────────────┘         └──────────┘
                            │
                            │  Event
                            ▼
                     ┌──────────┐
                     │ Notific. │
                     │ Service  │
                     └──────────┘
```

```python
# Event publishing
class OrderService:
    def __init__(self, event_publisher: EventPublisher):
        self.event_publisher = event_publisher
    
    def create_order(self, order: Order) -> Order:
        # Save order
        saved_order = self.repository.save(order)
        
        # Publish event (fire and forget)
        self.event_publisher.publish(
            topic="orders",
            event=OrderCreatedEvent(
                order_id=saved_order.id,
                customer_id=saved_order.customer_id,
                items=saved_order.items,
                total=saved_order.total,
                created_at=saved_order.created_at
            )
        )
        
        return saved_order

# Event consumer
class ShippingEventHandler:
    def handle_order_created(self, event: OrderCreatedEvent):
        # Create shipping label
        label = self.shipping_service.create_label(
            order_id=event.order_id,
            items=event.items
        )
        
        # Publish shipping created event
        self.event_publisher.publish(
            topic="shipping",
            event=ShippingLabelCreatedEvent(
                order_id=event.order_id,
                tracking_number=label.tracking_number
            )
        )
```

### Choosing Communication Style

| Scenario | Style | Reason |
|----------|-------|--------|
| Need immediate response | Sync | User waiting |
| Fire and forget | Async | No response needed |
| Long-running operation | Async | Don't block |
| Multiple consumers | Async | Fanout pattern |
| Strong consistency | Sync | Need confirmation |
| Loose coupling | Async | Services independent |

---

## API Design

### REST API Guidelines

```python
# Resource-based URLs
GET    /orders                 # List orders
POST   /orders                 # Create order
GET    /orders/{id}            # Get order
PUT    /orders/{id}            # Update order
DELETE /orders/{id}            # Delete order
GET    /orders/{id}/items      # Get order items
POST   /orders/{id}/cancel     # Cancel order (action)

# API versioning
GET /v1/orders
GET /v2/orders

# Standard response format
{
    "data": { ... },
    "meta": {
        "request_id": "req-123",
        "timestamp": "2024-01-15T10:30:00Z"
    }
}

# Error response
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid request",
        "details": [
            {"field": "email", "message": "Invalid format"}
        ]
    },
    "meta": {
        "request_id": "req-123"
    }
}
```

### gRPC for Internal Services

```protobuf
// order.proto
syntax = "proto3";
package orders;

service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (Order);
    rpc GetOrder(GetOrderRequest) returns (Order);
    rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
    rpc StreamOrderUpdates(StreamRequest) returns (stream OrderUpdate);
}

message CreateOrderRequest {
    string customer_id = 1;
    repeated OrderItem items = 2;
}

message Order {
    string id = 1;
    string customer_id = 2;
    repeated OrderItem items = 3;
    OrderStatus status = 4;
    Money total = 5;
    google.protobuf.Timestamp created_at = 6;
}

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_CONFIRMED = 2;
    ORDER_STATUS_SHIPPED = 3;
}
```

### API Gateway

```yaml
# Kong/AWS API Gateway configuration
routes:
  - path: /api/orders/*
    service: order-service
    plugins:
      - rate-limiting:
          minute: 100
      - jwt-auth:
          required: true
      - request-transformer:
          add:
            headers:
              - X-Request-ID: $(uuid)
  
  - path: /api/products/*
    service: product-service
    plugins:
      - caching:
          ttl: 300
```

---

## Data Management

### Database per Service

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ Order Service │    │ Customer Svc  │    │ Product Svc   │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        ▼                    ▼                    ▼
  ┌──────────┐         ┌──────────┐         ┌──────────┐
  │ Order DB │         │Customer  │         │Product DB│
  │(Postgres)│         │DB (Mongo)│         │(Postgres)│
  └──────────┘         └──────────┘         └──────────┘
```

**Rules:**
- Services never access other services' databases
- Data is replicated through events when needed
- Accept eventual consistency

### Saga Pattern (Distributed Transactions)

```python
# Choreography-based saga
class OrderSaga:
    """
    Order creation saga with compensation.
    
    Happy path:
    1. Create Order (pending)
    2. Reserve Inventory
    3. Process Payment
    4. Confirm Order
    
    Compensation (on failure):
    - Payment fails → Release Inventory → Cancel Order
    """
    
    def handle_order_created(self, event: OrderCreatedEvent):
        self.inventory_client.reserve(
            order_id=event.order_id,
            items=event.items
        )
    
    def handle_inventory_reserved(self, event: InventoryReservedEvent):
        self.payment_client.charge(
            order_id=event.order_id,
            amount=event.total
        )
    
    def handle_payment_completed(self, event: PaymentCompletedEvent):
        self.order_client.confirm(order_id=event.order_id)
    
    def handle_payment_failed(self, event: PaymentFailedEvent):
        # Compensation: release inventory
        self.inventory_client.release(order_id=event.order_id)
        self.order_client.cancel(
            order_id=event.order_id,
            reason="Payment failed"
        )
```

```python
# Orchestration-based saga
class OrderSagaOrchestrator:
    def execute(self, order: Order) -> SagaResult:
        saga_log = SagaLog(order_id=order.id)
        
        try:
            # Step 1: Create order
            self.order_service.create(order)
            saga_log.record("order_created")
            
            # Step 2: Reserve inventory
            self.inventory_service.reserve(order.id, order.items)
            saga_log.record("inventory_reserved")
            
            # Step 3: Process payment
            self.payment_service.charge(order.id, order.total)
            saga_log.record("payment_completed")
            
            # Step 4: Confirm order
            self.order_service.confirm(order.id)
            saga_log.record("order_confirmed")
            
            return SagaResult.success()
            
        except Exception as e:
            # Compensate in reverse order
            self._compensate(saga_log)
            return SagaResult.failure(str(e))
    
    def _compensate(self, saga_log: SagaLog):
        if saga_log.has("payment_completed"):
            self.payment_service.refund(saga_log.order_id)
        if saga_log.has("inventory_reserved"):
            self.inventory_service.release(saga_log.order_id)
        if saga_log.has("order_created"):
            self.order_service.cancel(saga_log.order_id)
```

### Event Sourcing

```python
# Event store
class EventStore:
    def append(self, aggregate_id: str, events: list[Event], expected_version: int):
        """Append events with optimistic concurrency."""
        current_version = self._get_version(aggregate_id)
        
        if current_version != expected_version:
            raise ConcurrencyError("Aggregate was modified")
        
        for i, event in enumerate(events):
            self.repository.insert(EventRecord(
                aggregate_id=aggregate_id,
                event_type=type(event).__name__,
                event_data=event.to_dict(),
                version=expected_version + i + 1,
                timestamp=datetime.utcnow()
            ))
    
    def get_events(self, aggregate_id: str) -> list[Event]:
        records = self.repository.find_by_aggregate(aggregate_id)
        return [self._deserialize(r) for r in records]

# Aggregate rebuilt from events
class Order:
    def __init__(self):
        self.id = None
        self.status = None
        self.items = []
        self._version = 0
    
    @classmethod
    def from_events(cls, events: list[Event]) -> "Order":
        order = cls()
        for event in events:
            order._apply(event)
        return order
    
    def _apply(self, event: Event):
        if isinstance(event, OrderCreated):
            self.id = event.order_id
            self.status = OrderStatus.PENDING
            self.items = event.items
        elif isinstance(event, OrderConfirmed):
            self.status = OrderStatus.CONFIRMED
        elif isinstance(event, OrderCancelled):
            self.status = OrderStatus.CANCELLED
        
        self._version += 1
```

---

## Resilience Patterns

### Circuit Breaker

```python
from circuitbreaker import circuit

class PaymentClient:
    @circuit(
        failure_threshold=5,      # Open after 5 failures
        recovery_timeout=30,      # Try again after 30s
        expected_exception=PaymentServiceError
    )
    def charge(self, amount: Money, order_id: str) -> PaymentResult:
        response = self.http_client.post(
            f"{self.base_url}/charges",
            json={"amount": amount, "order_id": order_id},
            timeout=5
        )
        return PaymentResult.from_response(response)
```

### Retry with Backoff

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type
)

class InventoryClient:
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception_type(TransientError)
    )
    def reserve(self, order_id: str, items: list) -> ReservationResult:
        return self._make_request(order_id, items)
```

### Bulkhead (Isolation)

```python
from concurrent.futures import ThreadPoolExecutor

class ServiceClients:
    def __init__(self):
        # Separate thread pools for different services
        self.payment_pool = ThreadPoolExecutor(max_workers=10, thread_name_prefix="payment")
        self.inventory_pool = ThreadPoolExecutor(max_workers=20, thread_name_prefix="inventory")
        self.notification_pool = ThreadPoolExecutor(max_workers=5, thread_name_prefix="notification")
    
    def call_payment(self, func, *args):
        """Payment calls isolated to their own pool."""
        future = self.payment_pool.submit(func, *args)
        return future.result(timeout=5)
```

### Timeout

```python
import httpx

client = httpx.Client(
    timeout=httpx.Timeout(
        connect=5.0,    # Connection timeout
        read=10.0,      # Read timeout
        write=5.0,      # Write timeout
        pool=2.0        # Pool timeout
    )
)
```

### Fallback

```python
class ProductService:
    def get_product(self, product_id: str) -> Product:
        try:
            return self.product_client.get(product_id)
        except ServiceUnavailableError:
            # Fallback to cache
            cached = self.cache.get(f"product:{product_id}")
            if cached:
                logger.warning(f"Using cached product {product_id}")
                return cached
            raise
```

---

## Service Discovery

### Client-Side Discovery

```python
# Using Consul
class ServiceDiscovery:
    def __init__(self, consul_client):
        self.consul = consul_client
        self._cache = {}
    
    def get_instances(self, service_name: str) -> list[ServiceInstance]:
        _, services = self.consul.health.service(service_name, passing=True)
        return [
            ServiceInstance(
                host=s["Service"]["Address"],
                port=s["Service"]["Port"]
            )
            for s in services
        ]
    
    def get_url(self, service_name: str) -> str:
        instances = self.get_instances(service_name)
        if not instances:
            raise NoHealthyInstanceError(service_name)
        # Round-robin or random selection
        instance = random.choice(instances)
        return f"http://{instance.host}:{instance.port}"
```

### Server-Side Discovery (Kubernetes)

```yaml
# Kubernetes Service
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
---
# Other services call: http://order-service/api/orders
```

---

## Observability

### Distributed Tracing

```python
from opentelemetry import trace
from opentelemetry.propagate import inject, extract

tracer = trace.get_tracer(__name__)

class OrderService:
    def create_order(self, request: Request, order: Order) -> Order:
        # Extract trace context from incoming request
        context = extract(request.headers)
        
        with tracer.start_as_current_span("create_order", context=context) as span:
            span.set_attribute("order.customer_id", order.customer_id)
            span.set_attribute("order.item_count", len(order.items))
            
            # Call inventory service (propagate context)
            headers = {}
            inject(headers)
            self.inventory_client.reserve(order, headers=headers)
            
            # Save order
            saved = self.repository.save(order)
            span.set_attribute("order.id", str(saved.id))
            
            return saved
```

### Structured Logging

```python
import structlog

logger = structlog.get_logger()

class OrderService:
    def create_order(self, order: Order) -> Order:
        log = logger.bind(
            service="order-service",
            order_id=str(order.id),
            customer_id=order.customer_id
        )
        
        log.info("Creating order", item_count=len(order.items))
        
        try:
            result = self._process_order(order)
            log.info("Order created", status=result.status)
            return result
        except Exception as e:
            log.error("Order creation failed", error=str(e))
            raise
```

### Health Checks

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health/live")
def liveness():
    """Am I running?"""
    return {"status": "ok"}

@app.get("/health/ready")
def readiness():
    """Am I ready to accept traffic?"""
    checks = {
        "database": check_database(),
        "cache": check_cache(),
        "downstream_services": check_downstream()
    }
    
    if all(checks.values()):
        return {"status": "ready", "checks": checks}
    else:
        return JSONResponse(
            status_code=503,
            content={"status": "not ready", "checks": checks}
        )
```

---

## Deployment

### Container Configuration

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY src/ ./src/

# Non-root user
RUN useradd -m appuser
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8080/health/live || exit 1

EXPOSE 8080

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: database-url
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

---

## Microservices Checklist

### Service Design
- [ ] Clear bounded context
- [ ] Single responsibility
- [ ] API versioned
- [ ] Own database

### Communication
- [ ] Sync for queries, async for commands
- [ ] Idempotent operations
- [ ] Event schema versioning
- [ ] Dead letter queues

### Resilience
- [ ] Circuit breakers
- [ ] Timeouts configured
- [ ] Retries with backoff
- [ ] Fallbacks defined

### Observability
- [ ] Distributed tracing
- [ ] Structured logging
- [ ] Health endpoints
- [ ] Metrics exposed

### Deployment
- [ ] Container health checks
- [ ] Resource limits
- [ ] Graceful shutdown
- [ ] Rolling updates
