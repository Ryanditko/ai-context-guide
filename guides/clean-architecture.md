# Clean Architecture

Guide for implementing Clean Architecture principles in software projects.

## Core Principles

### The Dependency Rule

Dependencies must point inward. Nothing in an inner circle can know about something in an outer circle.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Frameworks & Drivers                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     Interface Adapters                       │   │
│  │  ┌─────────────────────────────────────────────────────┐    │   │
│  │  │                 Application Layer                    │    │   │
│  │  │  ┌─────────────────────────────────────────────┐    │    │   │
│  │  │  │              Domain Layer                    │    │    │   │
│  │  │  │          (Entities & Business Rules)         │    │    │   │
│  │  │  └─────────────────────────────────────────────┘    │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Benefits

1. **Independence from Frameworks**: Business rules don't depend on specific frameworks
2. **Testability**: Business rules can be tested without UI, database, or external services
3. **Independence from UI**: UI can change without changing business rules
4. **Independence from Database**: Business rules don't depend on specific database
5. **Independence from External Agencies**: Business rules know nothing about external interfaces

---

## Layer Definitions

### Layer 1: Domain Layer (Entities)

The innermost layer containing enterprise-wide business rules.

#### Components

| Component | Purpose | Characteristics |
|-----------|---------|-----------------|
| **Entities** | Core business objects | Identity, state, business rules |
| **Value Objects** | Immutable descriptors | No identity, equality by value |
| **Domain Services** | Cross-entity operations | Stateless business logic |
| **Domain Events** | State change notifications | Immutable, past tense |
| **Aggregates** | Consistency boundaries | Transaction scope |
| **Repositories (Interface)** | Data access abstraction | Only interface definitions |

#### Entity Example

```python
# domain/entities/order.py

class Order:
    """Order entity with business rules and invariants."""
    
    def __init__(
        self,
        id: OrderId,
        customer_id: CustomerId,
        items: list[OrderItem],
        status: OrderStatus = OrderStatus.DRAFT
    ):
        self._id = id
        self._customer_id = customer_id
        self._items = list(items)
        self._status = status
        self._events: list[DomainEvent] = []
        
        self._validate_invariants()
    
    @property
    def id(self) -> OrderId:
        return self._id
    
    @property
    def total(self) -> Money:
        return sum((item.subtotal for item in self._items), Money.zero("USD"))
    
    def add_item(self, product: Product, quantity: int) -> None:
        """Add item to order. Domain rule: only draft orders can be modified."""
        if self._status != OrderStatus.DRAFT:
            raise OrderModificationError("Cannot modify non-draft order")
        
        if quantity <= 0:
            raise InvalidQuantityError("Quantity must be positive")
        
        existing = self._find_item(product.id)
        if existing:
            existing.increase_quantity(quantity)
        else:
            self._items.append(OrderItem(product, quantity))
    
    def submit(self) -> None:
        """Submit order for processing. Domain rule: must have at least one item."""
        if not self._items:
            raise EmptyOrderError("Cannot submit empty order")
        
        if self._status != OrderStatus.DRAFT:
            raise InvalidStateTransitionError(f"Cannot submit {self._status} order")
        
        self._status = OrderStatus.SUBMITTED
        self._events.append(OrderSubmittedEvent(self._id, self.total))
    
    def _validate_invariants(self) -> None:
        """Validate business invariants that must always hold."""
        if not self._customer_id:
            raise InvalidOrderError("Order must have a customer")
```

#### Value Object Example

```python
# domain/value_objects/money.py

@dataclass(frozen=True)
class Money:
    """Immutable value object representing monetary amount."""
    
    amount: Decimal
    currency: str
    
    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")
        if len(self.currency) != 3:
            raise ValueError("Currency must be 3-letter ISO code")
    
    def __add__(self, other: "Money") -> "Money":
        self._ensure_same_currency(other)
        return Money(self.amount + other.amount, self.currency)
    
    def __mul__(self, multiplier: int | Decimal) -> "Money":
        return Money(self.amount * Decimal(multiplier), self.currency)
    
    def _ensure_same_currency(self, other: "Money") -> None:
        if self.currency != other.currency:
            raise CurrencyMismatchError(f"Cannot operate on {self.currency} and {other.currency}")
    
    @classmethod
    def zero(cls, currency: str) -> "Money":
        return cls(Decimal("0"), currency)
```

#### Domain Service Example

```python
# domain/services/pricing_service.py

class PricingService:
    """Domain service for pricing calculations spanning multiple entities."""
    
    def calculate_order_total(
        self,
        items: list[OrderItem],
        discount: Discount | None = None,
        tax_rate: TaxRate | None = None
    ) -> OrderTotal:
        subtotal = sum((item.subtotal for item in items), Money.zero("USD"))
        
        discount_amount = Money.zero("USD")
        if discount:
            discount_amount = discount.apply(subtotal)
        
        taxable_amount = subtotal - discount_amount
        tax_amount = Money.zero("USD")
        if tax_rate:
            tax_amount = tax_rate.calculate(taxable_amount)
        
        return OrderTotal(
            subtotal=subtotal,
            discount=discount_amount,
            tax=tax_amount,
            total=taxable_amount + tax_amount
        )
```

---

### Layer 2: Application Layer (Use Cases)

Contains application-specific business rules. Orchestrates data flow to and from entities.

#### Components

| Component | Purpose | Characteristics |
|-----------|---------|-----------------|
| **Use Cases** | Application workflows | Orchestration, no business logic |
| **Commands/Queries** | Input DTOs | Immutable, validated |
| **Results** | Output DTOs | Success/failure with data |
| **Application Services** | Cross-cutting concerns | Transactions, authorization |
| **Port Interfaces** | Dependency contracts | Abstract definitions |

#### Use Case Example

```python
# application/use_cases/create_order.py

@dataclass(frozen=True)
class CreateOrderCommand:
    """Input DTO for creating an order."""
    customer_id: str
    items: list[OrderItemDTO]


@dataclass
class CreateOrderResult:
    """Output DTO with result."""
    success: bool
    order_id: str | None = None
    error: str | None = None
    
    @classmethod
    def ok(cls, order_id: str) -> "CreateOrderResult":
        return cls(success=True, order_id=order_id)
    
    @classmethod
    def fail(cls, error: str) -> "CreateOrderResult":
        return cls(success=False, error=error)


class CreateOrderUseCase:
    """
    Use case for creating a new order.
    Orchestrates the flow without containing business logic.
    """
    
    def __init__(
        self,
        customer_repository: CustomerRepository,
        product_repository: ProductRepository,
        order_repository: OrderRepository,
        event_publisher: EventPublisher,
        unit_of_work: UnitOfWork
    ):
        self._customer_repo = customer_repository
        self._product_repo = product_repository
        self._order_repo = order_repository
        self._event_publisher = event_publisher
        self._uow = unit_of_work
    
    def execute(self, command: CreateOrderCommand) -> CreateOrderResult:
        # 1. Validate customer exists
        customer = self._customer_repo.find_by_id(CustomerId(command.customer_id))
        if not customer:
            return CreateOrderResult.fail("Customer not found")
        
        # 2. Build order items (validate products exist)
        try:
            items = self._build_order_items(command.items)
        except ProductNotFoundError as e:
            return CreateOrderResult.fail(str(e))
        
        # 3. Create order (domain logic happens here)
        try:
            order = Order.create(
                id=OrderId.generate(),
                customer_id=customer.id,
                items=items
            )
        except DomainError as e:
            return CreateOrderResult.fail(str(e))
        
        # 4. Persist and publish events
        with self._uow:
            self._order_repo.save(order)
            for event in order.pull_events():
                self._event_publisher.publish(event)
            self._uow.commit()
        
        return CreateOrderResult.ok(str(order.id))
    
    def _build_order_items(self, item_dtos: list[OrderItemDTO]) -> list[OrderItem]:
        items = []
        for dto in item_dtos:
            product = self._product_repo.find_by_id(ProductId(dto.product_id))
            if not product:
                raise ProductNotFoundError(f"Product {dto.product_id} not found")
            items.append(OrderItem(product, dto.quantity))
        return items
```

#### Query Use Case Example

```python
# application/use_cases/get_order_details.py

@dataclass(frozen=True)
class GetOrderDetailsQuery:
    order_id: str
    include_customer: bool = False
    include_product_details: bool = False


@dataclass
class OrderDetailsDTO:
    id: str
    status: str
    total: str
    items: list[OrderItemDTO]
    customer: CustomerDTO | None = None
    created_at: datetime


class GetOrderDetailsUseCase:
    """Query use case - read-only, can bypass domain layer."""
    
    def __init__(
        self,
        order_read_model: OrderReadModel,
        customer_read_model: CustomerReadModel
    ):
        self._order_read_model = order_read_model
        self._customer_read_model = customer_read_model
    
    def execute(self, query: GetOrderDetailsQuery) -> OrderDetailsDTO | None:
        order = self._order_read_model.find_by_id(query.order_id)
        if not order:
            return None
        
        customer = None
        if query.include_customer:
            customer = self._customer_read_model.find_by_id(order.customer_id)
        
        return OrderDetailsDTO(
            id=order.id,
            status=order.status,
            total=str(order.total),
            items=order.items,
            customer=customer,
            created_at=order.created_at
        )
```

---

### Layer 3: Interface Adapters

Converts data between use cases/entities and external formats.

#### Components

| Component | Purpose | Characteristics |
|-----------|---------|-----------------|
| **Controllers** | Handle incoming requests | Convert request → command |
| **Presenters** | Format outgoing data | Convert result → response |
| **Gateways** | External service adapters | Implement port interfaces |
| **Repositories (Impl)** | Data persistence | Implement repository interfaces |
| **Mappers** | Data transformation | Entity ↔ DTO conversion |

#### Controller Example

```python
# adapters/controllers/order_controller.py

class OrderController:
    """HTTP controller - converts requests to use case commands."""
    
    def __init__(
        self,
        create_order: CreateOrderUseCase,
        get_order: GetOrderDetailsUseCase,
        presenter: OrderPresenter
    ):
        self._create_order = create_order
        self._get_order = get_order
        self._presenter = presenter
    
    def create(self, request: HttpRequest) -> HttpResponse:
        # 1. Parse and validate request
        try:
            body = self._parse_json(request.body)
            command = CreateOrderCommand(
                customer_id=body["customer_id"],
                items=[
                    OrderItemDTO(
                        product_id=item["product_id"],
                        quantity=item["quantity"]
                    )
                    for item in body["items"]
                ]
            )
        except (KeyError, ValueError) as e:
            return self._presenter.bad_request(str(e))
        
        # 2. Execute use case
        result = self._create_order.execute(command)
        
        # 3. Present result
        if result.success:
            return self._presenter.created(result.order_id)
        return self._presenter.error(result.error)
    
    def get(self, request: HttpRequest, order_id: str) -> HttpResponse:
        query = GetOrderDetailsQuery(
            order_id=order_id,
            include_customer="customer" in request.query.get("expand", [])
        )
        
        result = self._get_order.execute(query)
        
        if result is None:
            return self._presenter.not_found()
        return self._presenter.ok(result)
```

#### Repository Implementation Example

```python
# adapters/repositories/sql_order_repository.py

class SqlOrderRepository(OrderRepository):
    """SQL implementation of OrderRepository port."""
    
    def __init__(self, session: Session, mapper: OrderMapper):
        self._session = session
        self._mapper = mapper
    
    def find_by_id(self, order_id: OrderId) -> Order | None:
        record = self._session.query(OrderRecord).filter_by(id=str(order_id)).first()
        if not record:
            return None
        return self._mapper.to_entity(record)
    
    def save(self, order: Order) -> None:
        record = self._mapper.to_record(order)
        self._session.merge(record)
    
    def find_by_customer(
        self,
        customer_id: CustomerId,
        status: OrderStatus | None = None
    ) -> list[Order]:
        query = self._session.query(OrderRecord).filter_by(
            customer_id=str(customer_id)
        )
        if status:
            query = query.filter_by(status=status.value)
        
        return [self._mapper.to_entity(r) for r in query.all()]


class OrderMapper:
    """Maps between Order entity and database record."""
    
    def to_entity(self, record: OrderRecord) -> Order:
        return Order(
            id=OrderId(record.id),
            customer_id=CustomerId(record.customer_id),
            items=[self._item_to_entity(i) for i in record.items],
            status=OrderStatus(record.status),
            created_at=record.created_at
        )
    
    def to_record(self, order: Order) -> OrderRecord:
        return OrderRecord(
            id=str(order.id),
            customer_id=str(order.customer_id),
            items=[self._item_to_record(i) for i in order.items],
            status=order.status.value,
            total=float(order.total.amount),
            currency=order.total.currency,
            created_at=order.created_at
        )
```

#### Gateway Example

```python
# adapters/gateways/stripe_payment_gateway.py

class StripePaymentGateway(PaymentGateway):
    """Stripe implementation of PaymentGateway port."""
    
    def __init__(self, api_key: str, http_client: HttpClient):
        self._api_key = api_key
        self._http = http_client
    
    def charge(
        self,
        amount: Money,
        payment_method: PaymentMethod,
        idempotency_key: str
    ) -> PaymentResult:
        try:
            response = self._http.post(
                "https://api.stripe.com/v1/charges",
                headers={"Authorization": f"Bearer {self._api_key}"},
                json={
                    "amount": int(amount.amount * 100),  # Stripe uses cents
                    "currency": amount.currency.lower(),
                    "source": payment_method.token,
                },
                idempotency_key=idempotency_key
            )
            
            return PaymentResult.success(
                transaction_id=response["id"],
                amount=amount
            )
        
        except PaymentDeclinedError as e:
            return PaymentResult.declined(reason=e.decline_code)
        except HttpError as e:
            return PaymentResult.error(message=str(e))
```

---

### Layer 4: Frameworks & Drivers

The outermost layer with frameworks, tools, and external agencies.

#### Components

| Component | Purpose | Examples |
|-----------|---------|----------|
| **Web Framework** | HTTP handling | FastAPI, Flask, Express |
| **ORM** | Database mapping | SQLAlchemy, Prisma |
| **Message Queue** | Async messaging | RabbitMQ, Kafka clients |
| **External APIs** | Third-party services | Stripe SDK, AWS SDK |
| **Configuration** | App settings | Environment, config files |

#### Framework Composition Example

```python
# infrastructure/config/dependency_injection.py

def create_application() -> Application:
    """Compose all layers using dependency injection."""
    
    # Infrastructure
    config = load_config()
    db_session = create_db_session(config.database_url)
    http_client = HttpClient(timeout=30)
    event_bus = KafkaEventBus(config.kafka_brokers)
    
    # Adapters - Repositories
    order_mapper = OrderMapper()
    order_repository = SqlOrderRepository(db_session, order_mapper)
    customer_repository = SqlCustomerRepository(db_session)
    product_repository = SqlProductRepository(db_session)
    
    # Adapters - Gateways
    payment_gateway = StripePaymentGateway(config.stripe_key, http_client)
    
    # Application - Use Cases
    create_order = CreateOrderUseCase(
        customer_repository=customer_repository,
        product_repository=product_repository,
        order_repository=order_repository,
        event_publisher=event_bus,
        unit_of_work=db_session
    )
    
    get_order = GetOrderDetailsUseCase(
        order_read_model=OrderReadModel(db_session),
        customer_read_model=CustomerReadModel(db_session)
    )
    
    # Adapters - Controllers
    order_presenter = OrderPresenter()
    order_controller = OrderController(
        create_order=create_order,
        get_order=get_order,
        presenter=order_presenter
    )
    
    # Framework - Routes
    app = FastAPI()
    app.post("/orders")(order_controller.create)
    app.get("/orders/{order_id}")(order_controller.get)
    
    return app
```

---

## Project Structure

```
src/
├── domain/
│   ├── entities/
│   │   ├── order.py
│   │   ├── customer.py
│   │   └── product.py
│   ├── value_objects/
│   │   ├── money.py
│   │   ├── email.py
│   │   └── address.py
│   ├── services/
│   │   ├── pricing_service.py
│   │   └── inventory_service.py
│   ├── events/
│   │   ├── order_events.py
│   │   └── customer_events.py
│   ├── repositories/
│   │   ├── order_repository.py      # Interface only
│   │   └── customer_repository.py   # Interface only
│   └── exceptions.py
│
├── application/
│   ├── use_cases/
│   │   ├── orders/
│   │   │   ├── create_order.py
│   │   │   ├── cancel_order.py
│   │   │   └── get_order.py
│   │   └── customers/
│   │       ├── register_customer.py
│   │       └── update_profile.py
│   ├── services/
│   │   ├── notification_service.py
│   │   └── authorization_service.py
│   ├── ports/
│   │   ├── payment_gateway.py       # Interface
│   │   ├── email_sender.py          # Interface
│   │   └── event_publisher.py       # Interface
│   └── dtos/
│       ├── order_dtos.py
│       └── customer_dtos.py
│
├── adapters/
│   ├── controllers/
│   │   ├── order_controller.py
│   │   └── customer_controller.py
│   ├── presenters/
│   │   ├── order_presenter.py
│   │   └── error_presenter.py
│   ├── repositories/
│   │   ├── sql_order_repository.py
│   │   └── sql_customer_repository.py
│   ├── gateways/
│   │   ├── stripe_payment_gateway.py
│   │   └── sendgrid_email_sender.py
│   └── mappers/
│       ├── order_mapper.py
│       └── customer_mapper.py
│
├── infrastructure/
│   ├── config/
│   │   ├── settings.py
│   │   └── dependency_injection.py
│   ├── database/
│   │   ├── models.py
│   │   ├── migrations/
│   │   └── session.py
│   ├── messaging/
│   │   ├── kafka_producer.py
│   │   └── kafka_consumer.py
│   └── http/
│       └── http_client.py
│
└── main.py
```

---

## Testing Strategy by Layer

| Layer | Test Type | Dependencies | Focus |
|-------|-----------|--------------|-------|
| **Domain** | Unit | None (pure) | Business rules, invariants |
| **Application** | Unit | Mocked ports | Orchestration, flow |
| **Adapters** | Unit + Integration | Test doubles | Data transformation |
| **Infrastructure** | Integration | Real/containers | External systems |

---

## Common Mistakes to Avoid

| Mistake | Problem | Solution |
|---------|---------|----------|
| Business logic in controllers | Hard to test, duplicated | Move to domain/use cases |
| Entities depend on ORM | Framework lock-in | Map at adapter layer |
| Use cases return entities | Leaks domain to outer layers | Return DTOs |
| Domain imports infrastructure | Violates dependency rule | Use interfaces/ports |
| Fat use cases | Too much responsibility | Split into smaller use cases |
| Anemic domain model | Logic outside entities | Put behavior with data |

---

## Decision Guide

### When to Use Clean Architecture

**Good fit:**
- Large, complex domain
- Long-lived applications
- Multiple interfaces (API, CLI, UI)
- Team working on different layers
- High test coverage requirements

**May be overkill:**
- Simple CRUD applications
- Short-lived projects
- Prototypes/MVPs
- Small team, simple domain
