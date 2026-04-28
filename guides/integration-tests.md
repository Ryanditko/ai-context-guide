# Integration Tests

Comprehensive guide for writing tests that verify interaction between system components.

## Definition and Purpose

Integration tests verify that different modules, services, or systems work correctly when combined. They test the interfaces and interactions between components.

**Key differences from unit tests:**
- Test multiple components together
- Use real (or realistic) dependencies
- Verify data flows correctly between layers
- Slower but higher confidence

## Test Pyramid

```
                    ┌─────────┐
                   /           \        E2E Tests
                  /    ~10%     \       (UI, full system)
                 /               \
                ├─────────────────┤
               /                   \    Integration Tests
              /       ~20%          \   (API, DB, services)
             /                       \
            ├─────────────────────────┤
           /                           \    Unit Tests
          /           ~70%              \   (functions, classes)
         /                               \
        └─────────────────────────────────┘
```

## Types of Integration Tests

### 1. Component Integration

Tests interaction between internal application modules.

```python
# tests/integration/test_order_processing.py

class TestOrderProcessing:
    """Test integration between OrderService, InventoryService, and PaymentService."""
    
    @pytest.fixture
    def services(self, db_session):
        inventory_repo = SqlInventoryRepository(db_session)
        order_repo = SqlOrderRepository(db_session)
        
        inventory_service = InventoryService(inventory_repo)
        payment_service = PaymentService(MockPaymentGateway())
        order_service = OrderService(
            order_repo=order_repo,
            inventory_service=inventory_service,
            payment_service=payment_service,
        )
        
        return {
            "inventory": inventory_service,
            "orders": order_service,
        }
    
    def test_order_reduces_inventory(self, services, db_session):
        """When order is placed, inventory should be reduced."""
        # Arrange
        services["inventory"].add_stock("SKU-001", quantity=10)
        
        # Act
        order = services["orders"].create_order(
            customer_id="cust-123",
            items=[{"sku": "SKU-001", "quantity": 3}]
        )
        
        # Assert
        assert order.status == OrderStatus.CONFIRMED
        assert services["inventory"].get_stock("SKU-001") == 7
    
    def test_order_fails_when_insufficient_stock(self, services):
        """Order should fail if not enough inventory."""
        # Arrange
        services["inventory"].add_stock("SKU-001", quantity=2)
        
        # Act & Assert
        with pytest.raises(InsufficientStockError):
            services["orders"].create_order(
                customer_id="cust-123",
                items=[{"sku": "SKU-001", "quantity": 5}]
            )
    
    def test_inventory_restored_when_payment_fails(self, services):
        """Inventory should be restored if payment fails."""
        # Arrange
        services["inventory"].add_stock("SKU-001", quantity=10)
        services["orders"].payment_service.gateway.force_decline = True
        
        # Act
        with pytest.raises(PaymentDeclinedError):
            services["orders"].create_order(
                customer_id="cust-123",
                items=[{"sku": "SKU-001", "quantity": 3}]
            )
        
        # Assert - inventory should be restored
        assert services["inventory"].get_stock("SKU-001") == 10
```

### 2. Database Integration

Tests repository implementations with real database operations.

```python
# tests/integration/database/test_user_repository.py

@pytest.fixture(scope="function")
def db_session(test_database):
    """Provide transactional session that rolls back after each test."""
    connection = test_database.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    
    yield session
    
    session.close()
    transaction.rollback()
    connection.close()


class TestUserRepository:
    """Integration tests for UserRepository with real PostgreSQL."""
    
    def test_saves_and_retrieves_user(self, db_session):
        """User should be persisted with all attributes."""
        # Arrange
        repo = SqlUserRepository(db_session)
        user = User(
            id=UserId.generate(),
            email=Email("john@example.com"),
            name="John Doe",
            status=UserStatus.ACTIVE,
        )
        
        # Act
        repo.save(user)
        db_session.flush()
        
        # Clear session cache to force DB read
        db_session.expire_all()
        
        retrieved = repo.find_by_id(user.id)
        
        # Assert
        assert retrieved is not None
        assert retrieved.id == user.id
        assert retrieved.email.value == "john@example.com"
        assert retrieved.status == UserStatus.ACTIVE
    
    def test_finds_user_by_email(self, db_session):
        """Should find user by email address."""
        repo = SqlUserRepository(db_session)
        user = UserFactory.create(email=Email("unique@example.com"))
        repo.save(user)
        db_session.flush()
        
        found = repo.find_by_email(Email("unique@example.com"))
        
        assert found is not None
        assert found.id == user.id
    
    def test_returns_none_for_nonexistent_user(self, db_session):
        """Should return None when user doesn't exist."""
        repo = SqlUserRepository(db_session)
        
        result = repo.find_by_id(UserId("nonexistent-id"))
        
        assert result is None
    
    def test_updates_existing_user(self, db_session):
        """Should update user attributes."""
        repo = SqlUserRepository(db_session)
        user = UserFactory.create(name="Original Name")
        repo.save(user)
        db_session.flush()
        
        # Update
        user.name = "Updated Name"
        repo.save(user)
        db_session.flush()
        db_session.expire_all()
        
        # Verify
        retrieved = repo.find_by_id(user.id)
        assert retrieved.name == "Updated Name"
    
    def test_deletes_user(self, db_session):
        """Should delete user from database."""
        repo = SqlUserRepository(db_session)
        user = UserFactory.create()
        repo.save(user)
        db_session.flush()
        
        repo.delete(user.id)
        db_session.flush()
        
        assert repo.find_by_id(user.id) is None
    
    def test_pagination(self, db_session):
        """Should paginate results correctly."""
        repo = SqlUserRepository(db_session)
        
        # Create 25 users
        for i in range(25):
            repo.save(UserFactory.create())
        db_session.flush()
        
        # Get first page
        page1 = repo.find_all(page=1, per_page=10)
        assert len(page1.items) == 10
        assert page1.total == 25
        assert page1.has_next is True
        
        # Get second page
        page2 = repo.find_all(page=2, per_page=10)
        assert len(page2.items) == 10
        
        # Get last page
        page3 = repo.find_all(page=3, per_page=10)
        assert len(page3.items) == 5
        assert page3.has_next is False
    
    def test_complex_query_with_filters(self, db_session):
        """Should filter by multiple criteria."""
        repo = SqlUserRepository(db_session)
        
        # Create users with different attributes
        repo.save(UserFactory.create(status=UserStatus.ACTIVE, role=Role.ADMIN))
        repo.save(UserFactory.create(status=UserStatus.ACTIVE, role=Role.USER))
        repo.save(UserFactory.create(status=UserStatus.INACTIVE, role=Role.ADMIN))
        db_session.flush()
        
        # Filter active admins
        results = repo.find_by_criteria(
            status=UserStatus.ACTIVE,
            role=Role.ADMIN
        )
        
        assert len(results) == 1
        assert results[0].status == UserStatus.ACTIVE
        assert results[0].role == Role.ADMIN
```

### 3. API Integration

Tests HTTP endpoints with request/response cycle.

```python
# tests/integration/api/test_users_api.py

@pytest.fixture
def client(app):
    """Test client for HTTP requests."""
    return TestClient(app)


@pytest.fixture
def auth_headers(test_user):
    """Authentication headers for protected endpoints."""
    token = create_access_token(test_user.id)
    return {"Authorization": f"Bearer {token}"}


class TestUsersAPI:
    """Integration tests for /users endpoints."""
    
    # === CREATE ===
    
    def test_create_user_returns_201(self, client):
        """POST /users should create user and return 201."""
        payload = {
            "email": "newuser@example.com",
            "name": "New User",
            "password": "SecurePass123!"
        }
        
        response = client.post("/api/users", json=payload)
        
        assert response.status_code == 201
        data = response.json()
        assert "id" in data
        assert data["email"] == "newuser@example.com"
        assert "password" not in data  # Password not returned
    
    def test_create_user_validates_email(self, client):
        """POST /users with invalid email returns 400."""
        payload = {
            "email": "not-an-email",
            "name": "Test User",
            "password": "SecurePass123!"
        }
        
        response = client.post("/api/users", json=payload)
        
        assert response.status_code == 400
        assert "email" in response.json()["error"]["message"].lower()
    
    def test_create_user_prevents_duplicates(self, client, existing_user):
        """POST /users with existing email returns 409."""
        payload = {
            "email": existing_user.email,
            "name": "Another User",
            "password": "SecurePass123!"
        }
        
        response = client.post("/api/users", json=payload)
        
        assert response.status_code == 409
        assert response.json()["error"]["code"] == "EMAIL_EXISTS"
    
    # === READ ===
    
    def test_get_user_returns_details(self, client, auth_headers, test_user):
        """GET /users/{id} returns user details."""
        response = client.get(f"/api/users/{test_user.id}", headers=auth_headers)
        
        assert response.status_code == 200
        data = response.json()
        assert data["id"] == str(test_user.id)
        assert data["email"] == test_user.email
    
    def test_get_user_returns_404_for_nonexistent(self, client, auth_headers):
        """GET /users/{id} for nonexistent user returns 404."""
        response = client.get("/api/users/nonexistent-id", headers=auth_headers)
        
        assert response.status_code == 404
    
    def test_get_user_requires_authentication(self, client, test_user):
        """GET /users/{id} without auth returns 401."""
        response = client.get(f"/api/users/{test_user.id}")
        
        assert response.status_code == 401
    
    def test_list_users_with_pagination(self, client, auth_headers, db_session):
        """GET /users returns paginated list."""
        # Create 15 users
        for i in range(15):
            create_test_user(db_session, email=f"user{i}@test.com")
        
        response = client.get(
            "/api/users?page=1&per_page=10",
            headers=auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        assert len(data["data"]) == 10
        assert data["pagination"]["total"] >= 15
        assert data["pagination"]["page"] == 1
    
    # === UPDATE ===
    
    def test_update_user_returns_200(self, client, auth_headers, test_user):
        """PATCH /users/{id} updates user."""
        payload = {"name": "Updated Name"}
        
        response = client.patch(
            f"/api/users/{test_user.id}",
            json=payload,
            headers=auth_headers
        )
        
        assert response.status_code == 200
        assert response.json()["name"] == "Updated Name"
    
    def test_cannot_update_other_users(self, client, auth_headers, other_user):
        """PATCH /users/{id} for other user returns 403."""
        payload = {"name": "Hacked Name"}
        
        response = client.patch(
            f"/api/users/{other_user.id}",
            json=payload,
            headers=auth_headers
        )
        
        assert response.status_code == 403
    
    # === DELETE ===
    
    def test_delete_user_returns_204(self, client, admin_headers, test_user):
        """DELETE /users/{id} removes user."""
        response = client.delete(
            f"/api/users/{test_user.id}",
            headers=admin_headers
        )
        
        assert response.status_code == 204
        
        # Verify deleted
        get_response = client.get(
            f"/api/users/{test_user.id}",
            headers=admin_headers
        )
        assert get_response.status_code == 404
```

### 4. External Service Integration

Tests interaction with external APIs using mocks or test containers.

```python
# tests/integration/external/test_payment_gateway.py

@pytest.fixture
def wiremock(wiremock_container):
    """WireMock server for stubbing external APIs."""
    return WireMockClient(wiremock_container.get_url())


class TestStripePaymentGateway:
    """Integration tests for Stripe payment gateway."""
    
    def test_successful_charge(self, wiremock):
        """Successful charge returns transaction ID."""
        # Arrange - stub Stripe API
        wiremock.stub_for(
            post("/v1/charges")
            .with_header("Authorization", containing("Bearer"))
            .will_return(
                json_response({
                    "id": "ch_123456",
                    "status": "succeeded",
                    "amount": 10000,
                    "currency": "usd"
                })
            )
        )
        
        gateway = StripePaymentGateway(
            api_key="test_key",
            base_url=wiremock.url
        )
        
        # Act
        result = gateway.charge(
            amount=Money(Decimal("100.00"), "USD"),
            payment_method=PaymentMethod(token="tok_visa"),
            idempotency_key="order-123"
        )
        
        # Assert
        assert result.is_success
        assert result.transaction_id == "ch_123456"
    
    def test_declined_card(self, wiremock):
        """Declined card returns failure with reason."""
        wiremock.stub_for(
            post("/v1/charges")
            .will_return(
                json_response({
                    "error": {
                        "type": "card_error",
                        "code": "card_declined",
                        "decline_code": "insufficient_funds",
                        "message": "Your card has insufficient funds."
                    }
                }, status=402)
            )
        )
        
        gateway = StripePaymentGateway(api_key="test_key", base_url=wiremock.url)
        
        result = gateway.charge(
            amount=Money(Decimal("100.00"), "USD"),
            payment_method=PaymentMethod(token="tok_declined"),
            idempotency_key="order-456"
        )
        
        assert result.is_failure
        assert result.error_code == "card_declined"
        assert result.decline_reason == "insufficient_funds"
    
    def test_retries_on_network_error(self, wiremock):
        """Should retry on transient network failures."""
        # First two calls fail, third succeeds
        wiremock.stub_for(
            post("/v1/charges")
            .in_scenario("retry-scenario")
            .when_scenario_state_is("Started")
            .will_return(a_response().with_status(503))
            .will_set_state_to("First-Retry")
        )
        wiremock.stub_for(
            post("/v1/charges")
            .in_scenario("retry-scenario")
            .when_scenario_state_is("First-Retry")
            .will_return(a_response().with_status(503))
            .will_set_state_to("Second-Retry")
        )
        wiremock.stub_for(
            post("/v1/charges")
            .in_scenario("retry-scenario")
            .when_scenario_state_is("Second-Retry")
            .will_return(json_response({"id": "ch_789", "status": "succeeded"}))
        )
        
        gateway = StripePaymentGateway(api_key="test_key", base_url=wiremock.url)
        
        result = gateway.charge(
            amount=Money(Decimal("100.00"), "USD"),
            payment_method=PaymentMethod(token="tok_visa"),
            idempotency_key="order-789"
        )
        
        assert result.is_success
    
    def test_idempotency_key_sent(self, wiremock):
        """Idempotency key should be included in request."""
        wiremock.stub_for(
            post("/v1/charges")
            .with_header("Idempotency-Key", equal_to("order-unique-123"))
            .will_return(json_response({"id": "ch_123", "status": "succeeded"}))
        )
        
        gateway = StripePaymentGateway(api_key="test_key", base_url=wiremock.url)
        
        gateway.charge(
            amount=Money(Decimal("100.00"), "USD"),
            payment_method=PaymentMethod(token="tok_visa"),
            idempotency_key="order-unique-123"
        )
        
        # WireMock validates the header was present
        wiremock.verify(post_requested_for("/v1/charges"))
```

### 5. Message Queue Integration

Tests producers and consumers with real or containerized queues.

```python
# tests/integration/messaging/test_order_events.py

@pytest.fixture(scope="session")
def kafka_container():
    """Kafka container for integration tests."""
    with KafkaContainer("confluentinc/cp-kafka:7.4") as kafka:
        yield kafka


@pytest.fixture
def kafka_producer(kafka_container):
    return KafkaProducer(
        bootstrap_servers=kafka_container.get_bootstrap_server(),
        value_serializer=lambda v: json.dumps(v).encode()
    )


@pytest.fixture
def kafka_consumer(kafka_container):
    consumer = KafkaConsumer(
        "order-events",
        bootstrap_servers=kafka_container.get_bootstrap_server(),
        auto_offset_reset="earliest",
        value_deserializer=lambda m: json.loads(m.decode()),
        consumer_timeout_ms=10000
    )
    yield consumer
    consumer.close()


class TestOrderEventPublisher:
    """Test order event publishing to Kafka."""
    
    def test_publishes_order_created_event(self, kafka_producer, kafka_consumer):
        """OrderCreated event should be published to Kafka."""
        # Arrange
        publisher = KafkaEventPublisher(kafka_producer)
        event = OrderCreatedEvent(
            order_id="order-123",
            customer_id="cust-456",
            total=Decimal("99.99"),
            created_at=datetime.utcnow()
        )
        
        # Act
        publisher.publish(event)
        kafka_producer.flush()
        
        # Assert
        messages = list(kafka_consumer)
        assert len(messages) >= 1
        
        order_event = next(
            (m.value for m in messages if m.value.get("order_id") == "order-123"),
            None
        )
        assert order_event is not None
        assert order_event["event_type"] == "OrderCreated"
        assert order_event["customer_id"] == "cust-456"
    
    def test_consumer_processes_order_event(self, kafka_producer, db_session):
        """Consumer should process OrderCreated events."""
        # Arrange
        repo = SqlOrderReadModelRepository(db_session)
        consumer = OrderEventConsumer(repo)
        
        # Publish event
        kafka_producer.send("order-events", {
            "event_type": "OrderCreated",
            "order_id": "order-789",
            "customer_id": "cust-123",
            "total": "150.00",
            "created_at": datetime.utcnow().isoformat()
        })
        kafka_producer.flush()
        
        # Act - process messages
        consumer.process_messages(max_messages=1, timeout_ms=5000)
        
        # Assert - read model updated
        read_model = repo.find_by_id("order-789")
        assert read_model is not None
        assert read_model.customer_id == "cust-123"
```

---

## Contract Testing

Validate contracts between services using Pact or similar tools.

```python
# tests/contract/test_user_service_contract.py

import pact

@pytest.fixture
def pact_mock():
    """Pact mock server for contract testing."""
    pact = Pact(
        consumer="OrderService",
        provider="UserService",
        host_name="localhost",
        port=1234
    )
    pact.start_service()
    yield pact
    pact.stop_service()


class TestUserServiceContract:
    """Contract tests for UserService API."""
    
    def test_get_user_contract(self, pact_mock):
        """GET /users/{id} contract."""
        # Define expected interaction
        pact_mock.given(
            "user 123 exists"
        ).upon_receiving(
            "a request for user 123"
        ).with_request(
            method="GET",
            path="/users/123",
            headers={"Accept": "application/json"}
        ).will_respond_with(
            status=200,
            headers={"Content-Type": "application/json"},
            body={
                "id": "123",
                "email": Like("user@example.com"),
                "name": Like("John Doe"),
                "status": Term(r"active|inactive", "active")
            }
        )
        
        # Execute request against mock
        with pact_mock:
            client = UserServiceClient(base_url=pact_mock.uri)
            user = client.get_user("123")
            
            assert user.id == "123"
    
    def test_user_not_found_contract(self, pact_mock):
        """GET /users/{id} when user doesn't exist."""
        pact_mock.given(
            "user 999 does not exist"
        ).upon_receiving(
            "a request for nonexistent user 999"
        ).with_request(
            method="GET",
            path="/users/999"
        ).will_respond_with(
            status=404,
            body={
                "error": {
                    "code": "NOT_FOUND",
                    "message": Like("User not found")
                }
            }
        )
        
        with pact_mock:
            client = UserServiceClient(base_url=pact_mock.uri)
            
            with pytest.raises(UserNotFoundError):
                client.get_user("999")
```

---

## Test Environment Setup

### Testcontainers

```python
# tests/conftest.py

import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from testcontainers.kafka import KafkaContainer

@pytest.fixture(scope="session")
def postgres_container():
    """PostgreSQL container for tests."""
    with PostgresContainer("postgres:16") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def redis_container():
    """Redis container for tests."""
    with RedisContainer("redis:7") as redis:
        yield redis

@pytest.fixture(scope="session")
def kafka_container():
    """Kafka container for tests."""
    with KafkaContainer("confluentinc/cp-kafka:7.4") as kafka:
        yield kafka

@pytest.fixture(scope="function")
def db_session(postgres_container):
    """Database session with transaction rollback."""
    engine = create_engine(postgres_container.get_connection_url())
    
    # Run migrations
    run_migrations(engine)
    
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    
    yield session
    
    session.close()
    transaction.rollback()
    connection.close()
```

### Docker Compose for Tests

```yaml
# docker-compose.test.yml
version: "3.8"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: test_db
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5433:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test -d test_db"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    ports:
      - "6380:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  wiremock:
    image: wiremock/wiremock:3.0.0
    ports:
      - "8081:8080"
    volumes:
      - ./wiremock:/home/wiremock
```

---

## CI/CD Pipeline

```yaml
# .github/workflows/integration-tests.yml
name: Integration Tests

on: [push, pull_request]

jobs:
  integration-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      
      - name: Install dependencies
        run: pip install -r requirements-dev.txt
      
      - name: Run migrations
        run: alembic upgrade head
        env:
          DATABASE_URL: postgresql://postgres:test@localhost/test_db
      
      - name: Run integration tests
        run: pytest tests/integration -v --tb=short
        env:
          DATABASE_URL: postgresql://postgres:test@localhost/test_db
          REDIS_URL: redis://localhost:6379
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: integration-test-results
          path: test-results/
```

---

## Best Practices

### 1. Data Isolation

```python
# Each test should create and clean its own data
@pytest.fixture(autouse=True)
def clean_database(db_session):
    yield
    db_session.execute(text("TRUNCATE users, orders, products CASCADE"))
    db_session.commit()
```

### 2. Explicit Waits

```python
# Wait for async operations
from tenacity import retry, stop_after_delay, wait_fixed

@retry(stop=stop_after_delay(30), wait=wait_fixed(1))
def wait_for_order_processed(order_id: str) -> Order:
    order = order_repo.find(order_id)
    if order.status != OrderStatus.PROCESSED:
        raise AssertionError(f"Order not processed yet: {order.status}")
    return order
```

### 3. Test Tagging

```python
# Tag tests for selective execution
@pytest.mark.integration
@pytest.mark.slow
def test_full_checkout_flow():
    pass

@pytest.mark.integration
@pytest.mark.database
def test_complex_query():
    pass

# Run specific tests
# pytest -m "integration and not slow"
# pytest -m "database"
```

---

## Integration Test Checklist

### Before Writing
- [ ] Identified integration points
- [ ] Test environment configured
- [ ] Test data strategy defined
- [ ] External services mocked/containerized

### Test Quality
- [ ] Tests are independent
- [ ] Data is isolated per test
- [ ] Timeouts are explicit
- [ ] Error scenarios covered
- [ ] Assertions are meaningful

### CI/CD
- [ ] Tests run in pipeline
- [ ] Services properly started
- [ ] Reasonable timeout limits
- [ ] Flaky tests addressed
- [ ] Results are reported
