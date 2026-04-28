# Testing Strategy

Guide for designing comprehensive test suites with clear boundaries between test types.

## Test Pyramid

```
                    ┌─────────┐
                   /           \        E2E Tests
                  /    Few      \       (Slow, Expensive)
                 /               \
                ├─────────────────┤
               /                   \    Integration Tests
              /      Some           \   (Medium Speed)
             /                       \
            ├─────────────────────────┤
           /                           \    Unit Tests
          /          Many               \   (Fast, Cheap)
         /                               \
        └─────────────────────────────────┘
```

### Recommended Distribution

| Test Type | Percentage | Execution Time | Cost |
|-----------|------------|----------------|------|
| Unit | 70% | < 10s total | Low |
| Integration | 20% | < 5 min total | Medium |
| E2E | 10% | < 15 min total | High |

---

## Test Types Comparison

| Aspect | Unit | Integration | E2E |
|--------|------|-------------|-----|
| **Scope** | Single unit | Multiple components | Full system |
| **Dependencies** | Mocked | Real (some) | Real (all) |
| **Speed** | < 10ms | 100ms - 5s | 5s - 30s |
| **Reliability** | Very stable | Mostly stable | Can be flaky |
| **Feedback** | Immediate | Fast | Slow |
| **Debugging** | Easy | Moderate | Difficult |
| **Maintenance** | Low | Medium | High |

---

## Unit Tests

### Purpose

- Verify individual components in isolation
- Document expected behavior
- Enable fearless refactoring
- Provide fast feedback

### What to Unit Test

```
✅ Should Test:
├── Business logic
├── Algorithms
├── Data transformations
├── Validation rules
├── State machines
├── Edge cases
└── Error handling

❌ Should Not Test:
├── Private methods (test via public API)
├── Third-party libraries
├── Framework code
├── Trivial getters/setters
├── Configuration files
└── Database queries (integration test)
```

### Test Structure

```python
class TestOrderPricing:
    """Group tests by feature or behavior."""
    
    # === Setup ===
    
    @pytest.fixture
    def pricing_service(self):
        return PricingService()
    
    @pytest.fixture
    def standard_items(self):
        return [
            OrderItem(product_id="A", quantity=2, price=Money(50, "USD")),
            OrderItem(product_id="B", quantity=1, price=Money(30, "USD")),
        ]
    
    # === Happy Path ===
    
    def test_calculates_total_for_multiple_items(self, pricing_service, standard_items):
        """Total should be sum of all item subtotals."""
        # Arrange - done by fixtures
        
        # Act
        total = pricing_service.calculate_total(standard_items)
        
        # Assert
        assert total == Money(130, "USD")
    
    # === Edge Cases ===
    
    def test_returns_zero_for_empty_cart(self, pricing_service):
        """Empty cart should have zero total."""
        total = pricing_service.calculate_total([])
        assert total == Money(0, "USD")
    
    def test_handles_single_item(self, pricing_service):
        """Single item total equals item subtotal."""
        items = [OrderItem(product_id="A", quantity=1, price=Money(100, "USD"))]
        total = pricing_service.calculate_total(items)
        assert total == Money(100, "USD")
    
    # === Discount Rules ===
    
    def test_applies_percentage_discount(self, pricing_service, standard_items):
        """10% discount should reduce total by 10%."""
        discount = PercentageDiscount(10)
        
        total = pricing_service.calculate_total(standard_items, discount=discount)
        
        assert total == Money(117, "USD")  # 130 - 13
    
    def test_discount_cannot_exceed_total(self, pricing_service):
        """Fixed discount larger than total results in zero."""
        items = [OrderItem(product_id="A", quantity=1, price=Money(50, "USD"))]
        discount = FixedDiscount(Money(100, "USD"))
        
        total = pricing_service.calculate_total(items, discount=discount)
        
        assert total == Money(0, "USD")
    
    # === Error Cases ===
    
    def test_rejects_negative_quantities(self, pricing_service):
        """Negative quantities should raise error."""
        items = [OrderItem(product_id="A", quantity=-1, price=Money(50, "USD"))]
        
        with pytest.raises(InvalidQuantityError):
            pricing_service.calculate_total(items)
    
    def test_rejects_mixed_currencies(self, pricing_service):
        """Items with different currencies should raise error."""
        items = [
            OrderItem(product_id="A", quantity=1, price=Money(50, "USD")),
            OrderItem(product_id="B", quantity=1, price=Money(50, "EUR")),
        ]
        
        with pytest.raises(CurrencyMismatchError):
            pricing_service.calculate_total(items)
```

### Mocking Strategy

```python
# ✅ Good: Mock at boundaries (external dependencies)
def test_sends_notification_after_order_created(self, mocker):
    # Mock external dependency
    mock_notifier = mocker.Mock(spec=NotificationService)
    use_case = CreateOrderUseCase(notifier=mock_notifier)
    
    use_case.execute(valid_command)
    
    mock_notifier.send.assert_called_once()


# ❌ Bad: Mocking internal implementation details
def test_order_creation(self, mocker):
    # Don't mock domain logic
    mocker.patch.object(Order, '_validate_items')  # Too granular
    mocker.patch.object(PricingService, 'calculate')  # Implementation detail
```

---

## Integration Tests

### Purpose

- Verify components work together correctly
- Test real database queries
- Validate external service integration
- Ensure correct data transformation

### What to Integration Test

```
✅ Should Test:
├── Repository implementations
├── Database queries and transactions
├── API endpoints (request → response)
├── Message queue producers/consumers
├── External API clients
├── Cache implementations
└── File system operations

❌ Should Not Test:
├── Pure business logic (unit test)
├── UI rendering (E2E test)
├── Full user workflows (E2E test)
└── Third-party service internals
```

### Database Integration

```python
# tests/integration/repositories/test_order_repository.py

@pytest.fixture(scope="function")
def db_session(test_database):
    """Provide a transactional session that rolls back after each test."""
    connection = test_database.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    
    yield session
    
    session.close()
    transaction.rollback()
    connection.close()


class TestOrderRepository:
    """Integration tests for SQL OrderRepository."""
    
    def test_saves_and_retrieves_order(self, db_session):
        """Order should be persisted with all attributes."""
        # Arrange
        repository = SqlOrderRepository(db_session)
        order = OrderFactory.create(
            status=OrderStatus.PENDING,
            items=[
                OrderItemFactory.create(quantity=2, price=Money(50, "USD")),
                OrderItemFactory.create(quantity=1, price=Money(30, "USD")),
            ]
        )
        
        # Act
        repository.save(order)
        db_session.flush()
        retrieved = repository.find_by_id(order.id)
        
        # Assert
        assert retrieved is not None
        assert retrieved.id == order.id
        assert retrieved.status == OrderStatus.PENDING
        assert len(retrieved.items) == 2
        assert retrieved.total == Money(130, "USD")
    
    def test_finds_orders_by_customer(self, db_session):
        """Should return only orders for specified customer."""
        # Arrange
        repository = SqlOrderRepository(db_session)
        customer_id = CustomerId.generate()
        
        # Create orders for target customer
        for _ in range(3):
            order = OrderFactory.create(customer_id=customer_id)
            repository.save(order)
        
        # Create orders for other customer
        for _ in range(2):
            order = OrderFactory.create()
            repository.save(order)
        
        db_session.flush()
        
        # Act
        orders = repository.find_by_customer(customer_id)
        
        # Assert
        assert len(orders) == 3
        assert all(o.customer_id == customer_id for o in orders)
    
    def test_updates_order_status(self, db_session):
        """Status update should persist correctly."""
        # Arrange
        repository = SqlOrderRepository(db_session)
        order = OrderFactory.create(status=OrderStatus.PENDING)
        repository.save(order)
        db_session.flush()
        
        # Act
        order.confirm()
        repository.save(order)
        db_session.flush()
        
        # Retrieve fresh from database
        db_session.expire_all()
        retrieved = repository.find_by_id(order.id)
        
        # Assert
        assert retrieved.status == OrderStatus.CONFIRMED
```

### API Integration

```python
# tests/integration/api/test_orders_endpoint.py

@pytest.fixture
def client(test_app):
    """Test client for making HTTP requests."""
    return TestClient(test_app)


@pytest.fixture
def auth_headers(test_user):
    """Authentication headers for protected endpoints."""
    token = create_test_token(test_user.id)
    return {"Authorization": f"Bearer {token}"}


class TestOrdersEndpoint:
    """Integration tests for /orders API endpoints."""
    
    def test_create_order_returns_201(self, client, auth_headers, test_products):
        """POST /orders should create order and return 201."""
        # Arrange
        payload = {
            "items": [
                {"product_id": str(test_products[0].id), "quantity": 2},
                {"product_id": str(test_products[1].id), "quantity": 1},
            ]
        }
        
        # Act
        response = client.post("/orders", json=payload, headers=auth_headers)
        
        # Assert
        assert response.status_code == 201
        data = response.json()
        assert "id" in data
        assert data["status"] == "pending"
        assert len(data["items"]) == 2
    
    def test_create_order_validates_products(self, client, auth_headers):
        """POST /orders with invalid product should return 400."""
        payload = {
            "items": [
                {"product_id": "nonexistent-product", "quantity": 1},
            ]
        }
        
        response = client.post("/orders", json=payload, headers=auth_headers)
        
        assert response.status_code == 400
        assert "product" in response.json()["error"]["message"].lower()
    
    def test_get_order_returns_details(self, client, auth_headers, test_order):
        """GET /orders/{id} should return order details."""
        response = client.get(f"/orders/{test_order.id}", headers=auth_headers)
        
        assert response.status_code == 200
        data = response.json()
        assert data["id"] == str(test_order.id)
        assert "items" in data
        assert "total" in data
    
    def test_get_order_returns_404_for_missing(self, client, auth_headers):
        """GET /orders/{id} for nonexistent order returns 404."""
        response = client.get("/orders/nonexistent-id", headers=auth_headers)
        
        assert response.status_code == 404
    
    def test_unauthorized_access_returns_401(self, client, test_order):
        """GET /orders/{id} without auth returns 401."""
        response = client.get(f"/orders/{test_order.id}")
        
        assert response.status_code == 401
```

### External Service Integration

```python
# tests/integration/gateways/test_payment_gateway.py

@pytest.fixture
def wiremock(wiremock_container):
    """WireMock server for stubbing external APIs."""
    return WireMockClient(wiremock_container.get_url())


class TestStripePaymentGateway:
    """Integration tests for Stripe payment gateway."""
    
    def test_successful_charge(self, wiremock):
        """Successful charge should return transaction ID."""
        # Arrange
        wiremock.stub_for(
            post("/v1/charges")
            .will_return(
                json_response({
                    "id": "ch_123456",
                    "status": "succeeded",
                    "amount": 10000
                })
            )
        )
        
        gateway = StripePaymentGateway(
            api_key="test_key",
            base_url=wiremock.url
        )
        
        # Act
        result = gateway.charge(
            amount=Money(100, "USD"),
            payment_method=PaymentMethod(token="tok_visa"),
            idempotency_key="order-123"
        )
        
        # Assert
        assert result.is_success
        assert result.transaction_id == "ch_123456"
    
    def test_declined_card(self, wiremock):
        """Declined card should return failure with reason."""
        wiremock.stub_for(
            post("/v1/charges")
            .will_return(
                json_response({
                    "error": {
                        "type": "card_error",
                        "code": "card_declined",
                        "decline_code": "insufficient_funds"
                    }
                },
                status=402)
            )
        )
        
        gateway = StripePaymentGateway(api_key="test_key", base_url=wiremock.url)
        
        result = gateway.charge(
            amount=Money(100, "USD"),
            payment_method=PaymentMethod(token="tok_declined"),
            idempotency_key="order-456"
        )
        
        assert result.is_failure
        assert result.decline_reason == "insufficient_funds"
    
    def test_retries_on_network_error(self, wiremock):
        """Should retry on transient network failures."""
        wiremock.stub_for(
            post("/v1/charges")
            .in_scenario("retry")
            .when_scenario_state_is("Started")
            .will_return(a_response().with_status(503))
            .will_set_state_to("Retry1")
        )
        wiremock.stub_for(
            post("/v1/charges")
            .in_scenario("retry")
            .when_scenario_state_is("Retry1")
            .will_return(json_response({"id": "ch_789", "status": "succeeded"}))
        )
        
        gateway = StripePaymentGateway(api_key="test_key", base_url=wiremock.url)
        
        result = gateway.charge(
            amount=Money(100, "USD"),
            payment_method=PaymentMethod(token="tok_visa"),
            idempotency_key="order-789"
        )
        
        assert result.is_success
```

---

## End-to-End Tests

### Purpose

- Validate complete user journeys
- Ensure system works as a whole
- Catch integration issues between services
- Verify critical business flows

### What to E2E Test

```
✅ Should Test:
├── Critical user journeys (checkout, signup)
├── Authentication flows
├── Payment processing
├── Email notifications
├── Cross-service workflows
└── UI interactions

❌ Should Not Test:
├── Every possible scenario (unit test)
├── Internal implementation details
├── Performance (separate perf tests)
└── Exhaustive edge cases
```

### E2E Test Structure

```python
# tests/e2e/test_checkout_flow.py

@pytest.mark.e2e
class TestCheckoutFlow:
    """E2E tests for complete checkout journey."""
    
    def test_guest_checkout_complete_flow(self, browser, test_products):
        """Guest user can complete checkout from cart to confirmation."""
        # Step 1: Add items to cart
        browser.goto("/products")
        browser.click(f"[data-product-id='{test_products[0].id}'] .add-to-cart")
        browser.click(f"[data-product-id='{test_products[1].id}'] .add-to-cart")
        
        # Step 2: Go to cart
        browser.click(".cart-icon")
        assert browser.is_visible(".cart-item", count=2)
        
        # Step 3: Proceed to checkout
        browser.click(".checkout-button")
        
        # Step 4: Fill shipping info
        browser.fill("#shipping-name", "John Doe")
        browser.fill("#shipping-email", "john@example.com")
        browser.fill("#shipping-address", "123 Main St")
        browser.fill("#shipping-city", "New York")
        browser.fill("#shipping-zip", "10001")
        browser.click(".continue-to-payment")
        
        # Step 5: Fill payment info (test card)
        browser.fill("#card-number", "4242424242424242")
        browser.fill("#card-expiry", "12/25")
        browser.fill("#card-cvc", "123")
        
        # Step 6: Place order
        browser.click(".place-order")
        
        # Step 7: Verify confirmation
        browser.wait_for_selector(".order-confirmation")
        assert browser.text_content(".order-number") is not None
        assert "Thank you" in browser.text_content(".confirmation-message")
        
        # Step 8: Verify email sent
        email = wait_for_email("john@example.com", timeout=30)
        assert "Order Confirmation" in email.subject
    
    def test_checkout_handles_payment_failure(self, browser, test_products):
        """User sees clear error when payment fails."""
        # Setup: Add item and go to payment
        browser.goto(f"/products/{test_products[0].id}")
        browser.click(".add-to-cart")
        browser.goto("/checkout")
        fill_shipping_info(browser)
        browser.click(".continue-to-payment")
        
        # Use declined test card
        browser.fill("#card-number", "4000000000000002")  # Declined card
        browser.fill("#card-expiry", "12/25")
        browser.fill("#card-cvc", "123")
        browser.click(".place-order")
        
        # Verify error handling
        browser.wait_for_selector(".payment-error")
        assert "declined" in browser.text_content(".payment-error").lower()
        assert browser.is_visible(".try-again-button")
```

### E2E Best Practices

```python
# Helpers for readable E2E tests

def login_as(browser, user_type: str = "customer"):
    """Helper to login as specific user type."""
    credentials = TEST_USERS[user_type]
    browser.goto("/login")
    browser.fill("#email", credentials["email"])
    browser.fill("#password", credentials["password"])
    browser.click("#login-button")
    browser.wait_for_selector(".dashboard")


def add_product_to_cart(browser, product_id: str, quantity: int = 1):
    """Helper to add product to cart."""
    browser.goto(f"/products/{product_id}")
    browser.fill("#quantity", str(quantity))
    browser.click(".add-to-cart")
    browser.wait_for_selector(".cart-updated-toast")


def fill_shipping_info(browser, info: dict = None):
    """Helper to fill shipping form with default or custom info."""
    info = info or DEFAULT_SHIPPING_INFO
    for field, value in info.items():
        browser.fill(f"#shipping-{field}", value)
```

---

## Test Data Management

### Factories

```python
# tests/factories.py

class UserFactory(Factory):
    class Meta:
        model = User
    
    id = LazyAttribute(lambda _: UserId.generate())
    email = Sequence(lambda n: f"user{n}@example.com")
    name = Faker("name")
    status = UserStatus.ACTIVE
    
    class Params:
        admin = Trait(role=UserRole.ADMIN)
        inactive = Trait(status=UserStatus.INACTIVE)


class OrderFactory(Factory):
    class Meta:
        model = Order
    
    id = LazyAttribute(lambda _: OrderId.generate())
    customer = SubFactory(UserFactory)
    status = OrderStatus.PENDING
    
    @factory.lazy_attribute
    def items(self):
        return [OrderItemFactory() for _ in range(random.randint(1, 3))]
    
    @classmethod
    def with_total(cls, total: Money, **kwargs):
        """Create order with specific total."""
        item = OrderItemFactory(price=total, quantity=1)
        return cls(items=[item], **kwargs)
```

### Fixtures

```python
# tests/conftest.py

@pytest.fixture
def valid_user():
    """Provide a valid user for tests."""
    return UserFactory.create()


@pytest.fixture
def admin_user():
    """Provide an admin user for tests."""
    return UserFactory.create(admin=True)


@pytest.fixture
def completed_order(valid_user):
    """Provide a completed order for tests."""
    return OrderFactory.create(
        customer=valid_user,
        status=OrderStatus.COMPLETED
    )
```

---

## CI/CD Pipeline

```yaml
# .github/workflows/tests.yml

name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/unit -v --cov=src --cov-report=xml
      - uses: codecov/codecov-action@v4
    timeout-minutes: 5

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/integration -v
    timeout-minutes: 15

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
      - run: pip install -r requirements-dev.txt
      - run: playwright install
      - run: docker-compose up -d
      - run: pytest tests/e2e -v --headed=false
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: e2e-screenshots
          path: tests/e2e/screenshots/
    timeout-minutes: 30
```

---

## Metrics and Goals

| Metric | Target | Why |
|--------|--------|-----|
| Code coverage | > 80% | Ensure most code is exercised |
| Mutation score | > 70% | Verify tests catch bugs |
| Unit test time | < 10s | Fast feedback loop |
| Integration test time | < 5min | Reasonable CI time |
| E2E test time | < 15min | Don't block deployments |
| Flaky test rate | < 1% | Reliable CI pipeline |
| Test/code ratio | ~1:1 | Balanced test investment |
