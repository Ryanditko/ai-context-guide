# Unit Tests

Guide for writing effective, isolated, and maintainable unit tests across all layers of Clean Architecture.

## Fundamental Principles

### FIRST
- **Fast**: Tests should run quickly (< 10s for entire suite)
- **Isolated**: Each test is independent of others
- **Repeatable**: Same result in any environment
- **Self-validating**: Pass or fail, no manual interpretation
- **Timely**: Written alongside production code

### AAA Pattern
```
Arrange  → Set up preconditions and inputs
Act      → Execute the action being tested
Assert   → Verify the expected result
```

## Clean Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    External Layer                           │
│  (Frameworks, Drivers, UI, Database, External Services)     │
├─────────────────────────────────────────────────────────────┤
│                   Interface Adapters                        │
│  (Controllers, Gateways, Presenters, Repositories)          │
├─────────────────────────────────────────────────────────────┤
│                   Application Layer                         │
│  (Use Cases, Application Services, DTOs)                    │
├─────────────────────────────────────────────────────────────┤
│                     Domain Layer                            │
│  (Entities, Value Objects, Domain Services, Events)         │
└─────────────────────────────────────────────────────────────┘
```

## Project Structure

```
tests/
├── unit/
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── user_test.py
│   │   │   └── order_test.py
│   │   ├── value_objects/
│   │   │   ├── email_test.py
│   │   │   └── money_test.py
│   │   └── services/
│   │       └── pricing_service_test.py
│   ├── application/
│   │   ├── use_cases/
│   │   │   ├── create_user_test.py
│   │   │   └── process_order_test.py
│   │   └── services/
│   │       └── notification_service_test.py
│   ├── adapters/
│   │   ├── controllers/
│   │   │   └── user_controller_test.py
│   │   ├── repositories/
│   │   │   └── user_repository_test.py
│   │   └── presenters/
│   │       └── order_presenter_test.py
│   └── infrastructure/
│       ├── http/
│       │   └── http_client_test.py
│       └── cache/
│           └── redis_cache_test.py
├── fixtures/
│   ├── user_fixtures.py
│   ├── order_fixtures.py
│   └── factories.py
└── conftest.py
```

---

## Layer 1: Domain Layer Tests

The domain layer contains pure business logic. Tests here should be:
- **100% pure** (no mocks needed)
- **Fast** (no I/O)
- **Focused on business rules**

### 1.1 Entity Tests

```python
# tests/unit/domain/entities/user_test.py

class TestUser:
    """Tests for User entity business rules."""
    
    # === Identity and Creation ===
    
    def test_should_create_user_with_valid_data(self):
        # Arrange
        user_id = UserId.generate()
        email = Email("john@example.com")
        name = Name("John Doe")
        
        # Act
        user = User.create(id=user_id, email=email, name=name)
        
        # Assert
        assert user.id == user_id
        assert user.email == email
        assert user.name == name
        assert user.status == UserStatus.PENDING
        assert user.created_at is not None

    def test_should_reject_creation_with_invalid_email(self):
        # Arrange & Act & Assert
        with pytest.raises(InvalidEmailError) as exc_info:
            User.create(
                id=UserId.generate(),
                email=Email("invalid-email"),
                name=Name("John Doe")
            )
        assert "Invalid email format" in str(exc_info.value)

    # === State Transitions ===
    
    def test_should_activate_pending_user(self):
        # Arrange
        user = UserFactory.create(status=UserStatus.PENDING)
        
        # Act
        user.activate()
        
        # Assert
        assert user.status == UserStatus.ACTIVE
        assert user.activated_at is not None

    def test_should_not_activate_already_active_user(self):
        # Arrange
        user = UserFactory.create(status=UserStatus.ACTIVE)
        
        # Act & Assert
        with pytest.raises(InvalidStateTransitionError):
            user.activate()

    def test_should_not_activate_suspended_user(self):
        # Arrange
        user = UserFactory.create(status=UserStatus.SUSPENDED)
        
        # Act & Assert
        with pytest.raises(InvalidStateTransitionError) as exc_info:
            user.activate()
        assert "Cannot activate suspended user" in str(exc_info.value)

    # === Business Rules ===
    
    def test_should_calculate_account_age_in_days(self):
        # Arrange
        created_at = datetime(2024, 1, 1, tzinfo=UTC)
        user = UserFactory.create(created_at=created_at)
        current_date = datetime(2024, 1, 15, tzinfo=UTC)
        
        # Act
        age = user.account_age_days(as_of=current_date)
        
        # Assert
        assert age == 14

    def test_should_check_if_user_can_perform_action(self):
        # Arrange
        active_user = UserFactory.create(status=UserStatus.ACTIVE)
        pending_user = UserFactory.create(status=UserStatus.PENDING)
        
        # Act & Assert
        assert active_user.can_perform_purchases() is True
        assert pending_user.can_perform_purchases() is False

    # === Domain Events ===
    
    def test_should_emit_event_when_user_activated(self):
        # Arrange
        user = UserFactory.create(status=UserStatus.PENDING)
        
        # Act
        user.activate()
        
        # Assert
        events = user.pull_domain_events()
        assert len(events) == 1
        assert isinstance(events[0], UserActivatedEvent)
        assert events[0].user_id == user.id
```

### 1.2 Value Object Tests

```python
# tests/unit/domain/value_objects/email_test.py

class TestEmail:
    """Tests for Email value object."""
    
    # === Valid Cases ===
    
    @pytest.mark.parametrize("valid_email", [
        "user@example.com",
        "user.name@example.com",
        "user+tag@example.co.uk",
        "user@subdomain.example.com",
    ])
    def test_should_accept_valid_email_formats(self, valid_email):
        # Act
        email = Email(valid_email)
        
        # Assert
        assert email.value == valid_email.lower()

    # === Invalid Cases ===
    
    @pytest.mark.parametrize("invalid_email,expected_error", [
        ("", "Email cannot be empty"),
        ("invalid", "Invalid email format"),
        ("@example.com", "Invalid email format"),
        ("user@", "Invalid email format"),
        ("user@.com", "Invalid email format"),
        (None, "Email cannot be None"),
    ])
    def test_should_reject_invalid_email_formats(self, invalid_email, expected_error):
        # Act & Assert
        with pytest.raises(InvalidEmailError) as exc_info:
            Email(invalid_email)
        assert expected_error in str(exc_info.value)

    # === Equality ===
    
    def test_should_be_equal_when_same_value(self):
        # Arrange
        email1 = Email("user@example.com")
        email2 = Email("user@example.com")
        
        # Assert
        assert email1 == email2
        assert hash(email1) == hash(email2)

    def test_should_be_equal_case_insensitive(self):
        # Arrange
        email1 = Email("User@Example.com")
        email2 = Email("user@example.com")
        
        # Assert
        assert email1 == email2

    # === Behavior ===
    
    def test_should_extract_domain(self):
        # Arrange
        email = Email("user@example.com")
        
        # Act & Assert
        assert email.domain == "example.com"

    def test_should_extract_local_part(self):
        # Arrange
        email = Email("user@example.com")
        
        # Act & Assert
        assert email.local_part == "user"


# tests/unit/domain/value_objects/money_test.py

class TestMoney:
    """Tests for Money value object."""
    
    # === Creation ===
    
    def test_should_create_money_with_valid_amount(self):
        # Act
        money = Money(amount=Decimal("100.50"), currency="USD")
        
        # Assert
        assert money.amount == Decimal("100.50")
        assert money.currency == "USD"

    def test_should_reject_negative_amount(self):
        # Act & Assert
        with pytest.raises(InvalidMoneyError):
            Money(amount=Decimal("-10.00"), currency="USD")

    # === Arithmetic Operations ===
    
    def test_should_add_same_currency(self):
        # Arrange
        money1 = Money(Decimal("100.00"), "USD")
        money2 = Money(Decimal("50.00"), "USD")
        
        # Act
        result = money1.add(money2)
        
        # Assert
        assert result.amount == Decimal("150.00")
        assert result.currency == "USD"

    def test_should_not_add_different_currencies(self):
        # Arrange
        usd = Money(Decimal("100.00"), "USD")
        eur = Money(Decimal("50.00"), "EUR")
        
        # Act & Assert
        with pytest.raises(CurrencyMismatchError):
            usd.add(eur)

    def test_should_multiply_by_quantity(self):
        # Arrange
        unit_price = Money(Decimal("25.00"), "USD")
        
        # Act
        total = unit_price.multiply(4)
        
        # Assert
        assert total.amount == Decimal("100.00")

    # === Comparison ===
    
    def test_should_compare_amounts(self):
        # Arrange
        small = Money(Decimal("50.00"), "USD")
        large = Money(Decimal("100.00"), "USD")
        
        # Assert
        assert small < large
        assert large > small
        assert small <= large
        assert large >= small

    # === Formatting ===
    
    def test_should_format_for_display(self):
        # Arrange
        money = Money(Decimal("1234.56"), "USD")
        
        # Act & Assert
        assert money.format() == "$1,234.56"
```

### 1.3 Domain Service Tests

```python
# tests/unit/domain/services/pricing_service_test.py

class TestPricingService:
    """Tests for PricingService domain logic."""
    
    def test_should_calculate_total_without_discount(self):
        # Arrange
        service = PricingService()
        items = [
            OrderItem(product_id="P1", quantity=2, unit_price=Money(Decimal("50.00"), "USD")),
            OrderItem(product_id="P2", quantity=1, unit_price=Money(Decimal("30.00"), "USD")),
        ]
        
        # Act
        total = service.calculate_total(items)
        
        # Assert
        assert total.amount == Decimal("130.00")

    def test_should_apply_percentage_discount(self):
        # Arrange
        service = PricingService()
        subtotal = Money(Decimal("100.00"), "USD")
        discount = PercentageDiscount(10)  # 10%
        
        # Act
        total = service.apply_discount(subtotal, discount)
        
        # Assert
        assert total.amount == Decimal("90.00")

    def test_should_apply_fixed_discount(self):
        # Arrange
        service = PricingService()
        subtotal = Money(Decimal("100.00"), "USD")
        discount = FixedDiscount(Money(Decimal("15.00"), "USD"))
        
        # Act
        total = service.apply_discount(subtotal, discount)
        
        # Assert
        assert total.amount == Decimal("85.00")

    def test_should_not_apply_discount_greater_than_subtotal(self):
        # Arrange
        service = PricingService()
        subtotal = Money(Decimal("50.00"), "USD")
        discount = FixedDiscount(Money(Decimal("100.00"), "USD"))
        
        # Act
        total = service.apply_discount(subtotal, discount)
        
        # Assert
        assert total.amount == Decimal("0.00")  # Floor at zero

    def test_should_calculate_tax(self):
        # Arrange
        service = PricingService()
        subtotal = Money(Decimal("100.00"), "USD")
        tax_rate = TaxRate(Decimal("0.08"))  # 8%
        
        # Act
        tax = service.calculate_tax(subtotal, tax_rate)
        
        # Assert
        assert tax.amount == Decimal("8.00")
```

---

## Layer 2: Application Layer Tests

Application layer tests focus on use cases and orchestration. Mock dependencies at boundaries.

### 2.1 Use Case Tests

```python
# tests/unit/application/use_cases/create_user_test.py

class TestCreateUserUseCase:
    """Tests for CreateUser use case orchestration."""
    
    @pytest.fixture
    def mock_user_repository(self):
        return Mock(spec=UserRepository)
    
    @pytest.fixture
    def mock_email_service(self):
        return Mock(spec=EmailService)
    
    @pytest.fixture
    def mock_event_publisher(self):
        return Mock(spec=EventPublisher)
    
    @pytest.fixture
    def use_case(self, mock_user_repository, mock_email_service, mock_event_publisher):
        return CreateUserUseCase(
            user_repository=mock_user_repository,
            email_service=mock_email_service,
            event_publisher=mock_event_publisher,
        )
    
    # === Happy Path ===
    
    def test_should_create_user_successfully(
        self, use_case, mock_user_repository, mock_email_service, mock_event_publisher
    ):
        # Arrange
        command = CreateUserCommand(
            email="john@example.com",
            name="John Doe",
            password="SecurePass123!"
        )
        mock_user_repository.exists_by_email.return_value = False
        mock_user_repository.save.return_value = None
        
        # Act
        result = use_case.execute(command)
        
        # Assert
        assert result.is_success
        assert result.user_id is not None
        mock_user_repository.save.assert_called_once()
        mock_email_service.send_welcome_email.assert_called_once()
        mock_event_publisher.publish.assert_called_once()

    def test_should_hash_password_before_saving(
        self, use_case, mock_user_repository
    ):
        # Arrange
        command = CreateUserCommand(
            email="john@example.com",
            name="John Doe",
            password="SecurePass123!"
        )
        mock_user_repository.exists_by_email.return_value = False
        
        # Act
        use_case.execute(command)
        
        # Assert
        saved_user = mock_user_repository.save.call_args[0][0]
        assert saved_user.password_hash != "SecurePass123!"
        assert saved_user.password_hash.startswith("$argon2")

    # === Validation Errors ===
    
    def test_should_fail_when_email_already_exists(
        self, use_case, mock_user_repository, mock_email_service
    ):
        # Arrange
        command = CreateUserCommand(
            email="existing@example.com",
            name="John Doe",
            password="SecurePass123!"
        )
        mock_user_repository.exists_by_email.return_value = True
        
        # Act
        result = use_case.execute(command)
        
        # Assert
        assert result.is_failure
        assert result.error.code == "EMAIL_ALREADY_EXISTS"
        mock_user_repository.save.assert_not_called()
        mock_email_service.send_welcome_email.assert_not_called()

    def test_should_fail_when_password_too_weak(self, use_case):
        # Arrange
        command = CreateUserCommand(
            email="john@example.com",
            name="John Doe",
            password="weak"
        )
        
        # Act
        result = use_case.execute(command)
        
        # Assert
        assert result.is_failure
        assert result.error.code == "WEAK_PASSWORD"

    # === Edge Cases ===
    
    def test_should_normalize_email_before_checking(
        self, use_case, mock_user_repository
    ):
        # Arrange
        command = CreateUserCommand(
            email="  John@Example.COM  ",
            name="John Doe",
            password="SecurePass123!"
        )
        mock_user_repository.exists_by_email.return_value = False
        
        # Act
        use_case.execute(command)
        
        # Assert
        mock_user_repository.exists_by_email.assert_called_with("john@example.com")

    # === Error Handling ===
    
    def test_should_not_send_email_if_save_fails(
        self, use_case, mock_user_repository, mock_email_service
    ):
        # Arrange
        command = CreateUserCommand(
            email="john@example.com",
            name="John Doe",
            password="SecurePass123!"
        )
        mock_user_repository.exists_by_email.return_value = False
        mock_user_repository.save.side_effect = RepositoryError("Database unavailable")
        
        # Act
        result = use_case.execute(command)
        
        # Assert
        assert result.is_failure
        mock_email_service.send_welcome_email.assert_not_called()


# tests/unit/application/use_cases/process_order_test.py

class TestProcessOrderUseCase:
    """Tests for ProcessOrder use case with complex orchestration."""
    
    @pytest.fixture
    def dependencies(self):
        return {
            "order_repository": Mock(spec=OrderRepository),
            "inventory_service": Mock(spec=InventoryService),
            "payment_gateway": Mock(spec=PaymentGateway),
            "notification_service": Mock(spec=NotificationService),
            "event_publisher": Mock(spec=EventPublisher),
        }
    
    @pytest.fixture
    def use_case(self, dependencies):
        return ProcessOrderUseCase(**dependencies)
    
    def test_should_process_order_end_to_end(self, use_case, dependencies):
        # Arrange
        command = ProcessOrderCommand(
            order_id="order-123",
            payment_method="credit_card",
            payment_token="tok_visa"
        )
        
        order = OrderFactory.create(id="order-123", status=OrderStatus.PENDING)
        dependencies["order_repository"].find_by_id.return_value = order
        dependencies["inventory_service"].reserve_items.return_value = ReservationResult.success()
        dependencies["payment_gateway"].charge.return_value = PaymentResult.success(transaction_id="txn-456")
        
        # Act
        result = use_case.execute(command)
        
        # Assert
        assert result.is_success
        assert result.transaction_id == "txn-456"
        
        # Verify orchestration order
        call_order = []
        dependencies["order_repository"].find_by_id.assert_called_once()
        dependencies["inventory_service"].reserve_items.assert_called_once()
        dependencies["payment_gateway"].charge.assert_called_once()
        dependencies["notification_service"].send_order_confirmation.assert_called_once()

    def test_should_release_inventory_when_payment_fails(self, use_case, dependencies):
        # Arrange
        command = ProcessOrderCommand(
            order_id="order-123",
            payment_method="credit_card",
            payment_token="tok_declined"
        )
        
        order = OrderFactory.create(id="order-123")
        reservation = ReservationResult.success(reservation_id="res-789")
        
        dependencies["order_repository"].find_by_id.return_value = order
        dependencies["inventory_service"].reserve_items.return_value = reservation
        dependencies["payment_gateway"].charge.return_value = PaymentResult.failure("Card declined")
        
        # Act
        result = use_case.execute(command)
        
        # Assert
        assert result.is_failure
        dependencies["inventory_service"].release_reservation.assert_called_once_with("res-789")
        dependencies["notification_service"].send_order_confirmation.assert_not_called()
```

### 2.2 Application Service Tests

```python
# tests/unit/application/services/notification_service_test.py

class TestNotificationService:
    """Tests for NotificationService application logic."""
    
    @pytest.fixture
    def mock_email_provider(self):
        return Mock(spec=EmailProvider)
    
    @pytest.fixture
    def mock_sms_provider(self):
        return Mock(spec=SmsProvider)
    
    @pytest.fixture
    def mock_push_provider(self):
        return Mock(spec=PushNotificationProvider)
    
    @pytest.fixture
    def service(self, mock_email_provider, mock_sms_provider, mock_push_provider):
        return NotificationService(
            email_provider=mock_email_provider,
            sms_provider=mock_sms_provider,
            push_provider=mock_push_provider,
        )
    
    def test_should_send_notification_via_preferred_channel(
        self, service, mock_email_provider
    ):
        # Arrange
        user = UserFactory.create(
            notification_preferences=NotificationPreferences(
                preferred_channel=NotificationChannel.EMAIL
            )
        )
        notification = Notification(
            type=NotificationType.ORDER_SHIPPED,
            content="Your order has shipped!"
        )
        
        # Act
        service.send(user, notification)
        
        # Assert
        mock_email_provider.send.assert_called_once()

    def test_should_fallback_to_next_channel_on_failure(
        self, service, mock_email_provider, mock_sms_provider
    ):
        # Arrange
        user = UserFactory.create(
            notification_preferences=NotificationPreferences(
                preferred_channel=NotificationChannel.EMAIL,
                fallback_channel=NotificationChannel.SMS
            )
        )
        notification = Notification(
            type=NotificationType.ORDER_SHIPPED,
            content="Your order has shipped!"
        )
        mock_email_provider.send.side_effect = ProviderError("Service unavailable")
        
        # Act
        service.send(user, notification)
        
        # Assert
        mock_email_provider.send.assert_called_once()
        mock_sms_provider.send.assert_called_once()

    def test_should_respect_quiet_hours(self, service, mock_push_provider):
        # Arrange
        user = UserFactory.create(
            notification_preferences=NotificationPreferences(
                quiet_hours_start=time(22, 0),
                quiet_hours_end=time(8, 0)
            )
        )
        notification = Notification(
            type=NotificationType.MARKETING,
            content="Check out our sale!"
        )
        current_time = time(23, 0)  # Within quiet hours
        
        # Act
        result = service.send(user, notification, at_time=current_time)
        
        # Assert
        assert result.status == NotificationStatus.QUEUED
        mock_push_provider.send.assert_not_called()
```

---

## Layer 3: Interface Adapters Tests

Tests for controllers, presenters, and repositories focus on data transformation and protocol handling.

### 3.1 Controller Tests

```python
# tests/unit/adapters/controllers/user_controller_test.py

class TestUserController:
    """Tests for UserController request/response handling."""
    
    @pytest.fixture
    def mock_create_user_use_case(self):
        return Mock(spec=CreateUserUseCase)
    
    @pytest.fixture
    def mock_get_user_use_case(self):
        return Mock(spec=GetUserUseCase)
    
    @pytest.fixture
    def controller(self, mock_create_user_use_case, mock_get_user_use_case):
        return UserController(
            create_user=mock_create_user_use_case,
            get_user=mock_get_user_use_case,
        )
    
    # === Request Transformation ===
    
    def test_should_transform_request_to_command(
        self, controller, mock_create_user_use_case
    ):
        # Arrange
        request = CreateUserRequest(
            email="john@example.com",
            name="John Doe",
            password="SecurePass123!"
        )
        mock_create_user_use_case.execute.return_value = CreateUserResult.success(
            user_id="user-123"
        )
        
        # Act
        controller.create_user(request)
        
        # Assert
        command = mock_create_user_use_case.execute.call_args[0][0]
        assert isinstance(command, CreateUserCommand)
        assert command.email == "john@example.com"
        assert command.name == "John Doe"

    # === Response Transformation ===
    
    def test_should_transform_success_result_to_response(
        self, controller, mock_create_user_use_case
    ):
        # Arrange
        request = CreateUserRequest(
            email="john@example.com",
            name="John Doe",
            password="SecurePass123!"
        )
        mock_create_user_use_case.execute.return_value = CreateUserResult.success(
            user_id="user-123"
        )
        
        # Act
        response = controller.create_user(request)
        
        # Assert
        assert response.status_code == 201
        assert response.body["id"] == "user-123"
        assert "password" not in response.body

    def test_should_transform_validation_error_to_400(
        self, controller, mock_create_user_use_case
    ):
        # Arrange
        request = CreateUserRequest(
            email="invalid-email",
            name="John Doe",
            password="weak"
        )
        mock_create_user_use_case.execute.return_value = CreateUserResult.failure(
            error=ValidationError(
                code="VALIDATION_ERROR",
                details=[
                    {"field": "email", "message": "Invalid format"},
                    {"field": "password", "message": "Too weak"},
                ]
            )
        )
        
        # Act
        response = controller.create_user(request)
        
        # Assert
        assert response.status_code == 400
        assert response.body["error"]["code"] == "VALIDATION_ERROR"
        assert len(response.body["error"]["details"]) == 2

    def test_should_transform_conflict_error_to_409(
        self, controller, mock_create_user_use_case
    ):
        # Arrange
        request = CreateUserRequest(
            email="existing@example.com",
            name="John Doe",
            password="SecurePass123!"
        )
        mock_create_user_use_case.execute.return_value = CreateUserResult.failure(
            error=ConflictError(code="EMAIL_ALREADY_EXISTS")
        )
        
        # Act
        response = controller.create_user(request)
        
        # Assert
        assert response.status_code == 409

    # === Input Validation ===
    
    def test_should_reject_missing_required_fields(self, controller):
        # Arrange
        request = CreateUserRequest(
            email=None,
            name="John Doe",
            password="SecurePass123!"
        )
        
        # Act
        response = controller.create_user(request)
        
        # Assert
        assert response.status_code == 400
        assert "email" in response.body["error"]["message"].lower()
```

### 3.2 Repository Tests

```python
# tests/unit/adapters/repositories/user_repository_test.py

class TestUserRepository:
    """Tests for UserRepository data transformation."""
    
    @pytest.fixture
    def mock_database(self):
        return Mock(spec=Database)
    
    @pytest.fixture
    def repository(self, mock_database):
        return UserRepository(database=mock_database)
    
    # === Save Operations ===
    
    def test_should_transform_entity_to_database_record(
        self, repository, mock_database
    ):
        # Arrange
        user = User(
            id=UserId("user-123"),
            email=Email("john@example.com"),
            name=Name("John Doe"),
            status=UserStatus.ACTIVE,
            created_at=datetime(2024, 1, 15, tzinfo=UTC)
        )
        
        # Act
        repository.save(user)
        
        # Assert
        saved_record = mock_database.insert.call_args[0][1]
        assert saved_record["id"] == "user-123"
        assert saved_record["email"] == "john@example.com"
        assert saved_record["status"] == "active"
        assert saved_record["created_at"] == "2024-01-15T00:00:00+00:00"

    # === Find Operations ===
    
    def test_should_transform_database_record_to_entity(
        self, repository, mock_database
    ):
        # Arrange
        mock_database.find_one.return_value = {
            "id": "user-123",
            "email": "john@example.com",
            "name": "John Doe",
            "status": "active",
            "created_at": "2024-01-15T00:00:00+00:00"
        }
        
        # Act
        user = repository.find_by_id(UserId("user-123"))
        
        # Assert
        assert isinstance(user, User)
        assert user.id.value == "user-123"
        assert user.email.value == "john@example.com"
        assert user.status == UserStatus.ACTIVE

    def test_should_return_none_when_not_found(
        self, repository, mock_database
    ):
        # Arrange
        mock_database.find_one.return_value = None
        
        # Act
        result = repository.find_by_id(UserId("nonexistent"))
        
        # Assert
        assert result is None

    # === Query Building ===
    
    def test_should_build_correct_query_for_email_search(
        self, repository, mock_database
    ):
        # Arrange
        mock_database.find_one.return_value = None
        
        # Act
        repository.find_by_email(Email("john@example.com"))
        
        # Assert
        query = mock_database.find_one.call_args[0][1]
        assert query == {"email": "john@example.com"}

    def test_should_handle_pagination_parameters(
        self, repository, mock_database
    ):
        # Arrange
        mock_database.find_many.return_value = []
        
        # Act
        repository.find_all(page=2, per_page=20)
        
        # Assert
        call_kwargs = mock_database.find_many.call_args[1]
        assert call_kwargs["skip"] == 20
        assert call_kwargs["limit"] == 20
```

### 3.3 Presenter Tests

```python
# tests/unit/adapters/presenters/order_presenter_test.py

class TestOrderPresenter:
    """Tests for OrderPresenter output formatting."""
    
    @pytest.fixture
    def presenter(self):
        return OrderPresenter()
    
    def test_should_format_order_for_api_response(self, presenter):
        # Arrange
        order = Order(
            id=OrderId("order-123"),
            customer_id=CustomerId("cust-456"),
            items=[
                OrderItem(product_id="prod-1", quantity=2, unit_price=Money(Decimal("50.00"), "USD")),
                OrderItem(product_id="prod-2", quantity=1, unit_price=Money(Decimal("30.00"), "USD")),
            ],
            status=OrderStatus.CONFIRMED,
            total=Money(Decimal("130.00"), "USD"),
            created_at=datetime(2024, 1, 15, 10, 30, 0, tzinfo=UTC)
        )
        
        # Act
        result = presenter.to_api_response(order)
        
        # Assert
        assert result == {
            "id": "order-123",
            "customer_id": "cust-456",
            "items": [
                {"product_id": "prod-1", "quantity": 2, "unit_price": "50.00"},
                {"product_id": "prod-2", "quantity": 1, "unit_price": "30.00"},
            ],
            "status": "confirmed",
            "total": "130.00",
            "currency": "USD",
            "created_at": "2024-01-15T10:30:00Z"
        }

    def test_should_format_order_list_with_pagination(self, presenter):
        # Arrange
        orders = [OrderFactory.create() for _ in range(2)]
        pagination = Pagination(total=100, page=2, per_page=20)
        
        # Act
        result = presenter.to_list_response(orders, pagination)
        
        # Assert
        assert len(result["data"]) == 2
        assert result["pagination"]["total"] == 100
        assert result["pagination"]["page"] == 2
        assert result["pagination"]["total_pages"] == 5

    def test_should_include_links_for_hateoas(self, presenter):
        # Arrange
        order = OrderFactory.create(id="order-123", status=OrderStatus.PENDING)
        
        # Act
        result = presenter.to_api_response(order, include_links=True)
        
        # Assert
        assert "_links" in result
        assert result["_links"]["self"]["href"] == "/orders/order-123"
        assert "cancel" in result["_links"]
        assert "pay" in result["_links"]

    def test_should_hide_sensitive_fields(self, presenter):
        # Arrange
        order = OrderFactory.create(
            payment_details=PaymentDetails(
                card_last_four="4242",
                card_token="tok_secret_123"
            )
        )
        
        # Act
        result = presenter.to_api_response(order)
        
        # Assert
        assert "card_last_four" in result.get("payment", {})
        assert "card_token" not in result.get("payment", {})
```

---

## Layer 4: Infrastructure Layer Tests

Infrastructure tests focus on integration with external systems. Use test doubles or containers.

### 4.1 HTTP Client Tests

```python
# tests/unit/infrastructure/http/http_client_test.py

class TestHttpClient:
    """Tests for HTTP client wrapper."""
    
    @pytest.fixture
    def mock_session(self):
        return Mock(spec=requests.Session)
    
    @pytest.fixture
    def client(self, mock_session):
        return HttpClient(
            session=mock_session,
            base_url="https://api.example.com",
            timeout=30
        )
    
    # === Request Building ===
    
    def test_should_add_default_headers(self, client, mock_session):
        # Arrange
        mock_session.request.return_value = Mock(
            status_code=200,
            json=lambda: {"data": "test"}
        )
        
        # Act
        client.get("/users")
        
        # Assert
        call_kwargs = mock_session.request.call_args[1]
        assert "Content-Type" in call_kwargs["headers"]
        assert "User-Agent" in call_kwargs["headers"]

    def test_should_add_authentication_header(self, client, mock_session):
        # Arrange
        mock_session.request.return_value = Mock(status_code=200, json=lambda: {})
        client.set_auth_token("bearer-token-123")
        
        # Act
        client.get("/users")
        
        # Assert
        call_kwargs = mock_session.request.call_args[1]
        assert call_kwargs["headers"]["Authorization"] == "Bearer bearer-token-123"

    # === Response Handling ===
    
    def test_should_parse_json_response(self, client, mock_session):
        # Arrange
        mock_session.request.return_value = Mock(
            status_code=200,
            json=lambda: {"id": "123", "name": "Test"}
        )
        
        # Act
        response = client.get("/users/123")
        
        # Assert
        assert response.data["id"] == "123"

    def test_should_raise_on_4xx_errors(self, client, mock_session):
        # Arrange
        mock_session.request.return_value = Mock(
            status_code=404,
            json=lambda: {"error": "Not found"}
        )
        
        # Act & Assert
        with pytest.raises(HttpClientError) as exc_info:
            client.get("/users/nonexistent")
        assert exc_info.value.status_code == 404

    # === Retry Logic ===
    
    def test_should_retry_on_transient_errors(self, client, mock_session):
        # Arrange
        mock_session.request.side_effect = [
            Mock(status_code=503),
            Mock(status_code=503),
            Mock(status_code=200, json=lambda: {"success": True})
        ]
        
        # Act
        response = client.get("/users", retry_count=3)
        
        # Assert
        assert response.data["success"] is True
        assert mock_session.request.call_count == 3

    def test_should_not_retry_on_client_errors(self, client, mock_session):
        # Arrange
        mock_session.request.return_value = Mock(
            status_code=400,
            json=lambda: {"error": "Bad request"}
        )
        
        # Act & Assert
        with pytest.raises(HttpClientError):
            client.get("/users", retry_count=3)
        assert mock_session.request.call_count == 1  # No retries
```

### 4.2 Cache Tests

```python
# tests/unit/infrastructure/cache/redis_cache_test.py

class TestRedisCache:
    """Tests for Redis cache wrapper."""
    
    @pytest.fixture
    def mock_redis(self):
        return Mock(spec=Redis)
    
    @pytest.fixture
    def cache(self, mock_redis):
        return RedisCache(client=mock_redis, default_ttl=300)
    
    # === Basic Operations ===
    
    def test_should_serialize_value_before_storing(self, cache, mock_redis):
        # Arrange
        data = {"user_id": "123", "name": "John"}
        
        # Act
        cache.set("user:123", data)
        
        # Assert
        mock_redis.setex.assert_called_once()
        stored_value = mock_redis.setex.call_args[0][2]
        assert json.loads(stored_value) == data

    def test_should_deserialize_value_when_retrieving(self, cache, mock_redis):
        # Arrange
        mock_redis.get.return_value = '{"user_id": "123", "name": "John"}'
        
        # Act
        result = cache.get("user:123")
        
        # Assert
        assert result == {"user_id": "123", "name": "John"}

    def test_should_return_none_for_missing_key(self, cache, mock_redis):
        # Arrange
        mock_redis.get.return_value = None
        
        # Act
        result = cache.get("nonexistent")
        
        # Assert
        assert result is None

    # === TTL Handling ===
    
    def test_should_use_default_ttl(self, cache, mock_redis):
        # Act
        cache.set("key", "value")
        
        # Assert
        mock_redis.setex.assert_called_with("key", 300, ANY)

    def test_should_allow_custom_ttl(self, cache, mock_redis):
        # Act
        cache.set("key", "value", ttl=60)
        
        # Assert
        mock_redis.setex.assert_called_with("key", 60, ANY)

    # === Cache Patterns ===
    
    def test_should_implement_get_or_set_pattern(self, cache, mock_redis):
        # Arrange
        mock_redis.get.return_value = None
        factory = Mock(return_value={"computed": "value"})
        
        # Act
        result = cache.get_or_set("key", factory, ttl=60)
        
        # Assert
        assert result == {"computed": "value"}
        factory.assert_called_once()
        mock_redis.setex.assert_called_once()

    def test_should_not_call_factory_when_cached(self, cache, mock_redis):
        # Arrange
        mock_redis.get.return_value = '{"cached": "value"}'
        factory = Mock()
        
        # Act
        result = cache.get_or_set("key", factory)
        
        # Assert
        assert result == {"cached": "value"}
        factory.assert_not_called()
```

---

## Test Fixtures and Factories

### Shared Fixtures

```python
# tests/conftest.py

import pytest
from datetime import datetime, UTC

@pytest.fixture
def freeze_time():
    """Fixture to freeze time for deterministic tests."""
    frozen = datetime(2024, 1, 15, 10, 30, 0, tzinfo=UTC)
    with freeze_time(frozen):
        yield frozen

@pytest.fixture
def random_seed():
    """Fixture for deterministic random values."""
    import random
    random.seed(42)
    yield
    random.seed()
```

### Factories

```python
# tests/fixtures/factories.py

from factory import Factory, Faker, LazyAttribute, SubFactory
from uuid import uuid4

class UserFactory(Factory):
    class Meta:
        model = User
    
    id = LazyAttribute(lambda _: UserId(str(uuid4())))
    email = LazyAttribute(lambda obj: Email(f"user-{obj.id.value[:8]}@example.com"))
    name = Faker("name")
    status = UserStatus.ACTIVE
    created_at = LazyAttribute(lambda _: datetime.now(UTC))

    class Params:
        pending = Trait(status=UserStatus.PENDING)
        suspended = Trait(status=UserStatus.SUSPENDED)


class OrderFactory(Factory):
    class Meta:
        model = Order
    
    id = LazyAttribute(lambda _: OrderId(str(uuid4())))
    customer = SubFactory(UserFactory)
    status = OrderStatus.PENDING
    items = []
    
    @classmethod
    def with_items(cls, count=3, **kwargs):
        items = [OrderItemFactory.create() for _ in range(count)]
        return cls.create(items=items, **kwargs)


class OrderItemFactory(Factory):
    class Meta:
        model = OrderItem
    
    product_id = LazyAttribute(lambda _: f"prod-{uuid4().hex[:8]}")
    quantity = 1
    unit_price = LazyAttribute(lambda _: Money(Decimal("25.00"), "USD"))
```

---

## Testing Patterns Summary

| Layer | What to Test | Mocking Strategy |
|-------|-------------|------------------|
| **Domain** | Business rules, invariants, state transitions | No mocks (pure) |
| **Application** | Orchestration, flow, error handling | Mock all boundaries |
| **Adapters** | Data transformation, protocol handling | Mock infrastructure |
| **Infrastructure** | External integration, serialization | Mock external clients |

## Coverage Guidelines

| Component | Minimum Coverage | Focus Areas |
|-----------|-----------------|-------------|
| Domain Entities | 95%+ | All business rules |
| Value Objects | 100% | Validation, equality |
| Use Cases | 90%+ | Happy path + errors |
| Controllers | 85%+ | Request/response mapping |
| Repositories | 80%+ | Query building, mapping |
| Infrastructure | 70%+ | Error handling, retries |

## Anti-patterns to Avoid

| Anti-pattern | Problem | Solution |
|--------------|---------|----------|
| Testing implementation | Breaks on refactoring | Test behavior |
| Excessive mocking | Fragile tests | Mock only boundaries |
| Test interdependence | Order-dependent failures | Complete isolation |
| Slow tests | Slow feedback | Keep domain pure |
| Testing private methods | Coupling to internals | Test via public API |
| Assert per line of code | Meaningless coverage | Assert per behavior |
