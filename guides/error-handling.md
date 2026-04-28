# Error Handling

Guide for consistent and robust error handling in applications.

## Fundamental Principles

### 1. Fail Fast
Detect and report errors as early as possible.

```python
def process_payment(amount: float, card_token: str):
    # Validation at the beginning
    if amount <= 0:
        raise ValueError("Amount must be positive")
    if not card_token:
        raise ValueError("Card token is required")
    
    # Main logic only executes if inputs are valid
    return payment_gateway.charge(amount, card_token)
```

### 2. Fail Gracefully
When failing, fail in a controlled manner.

```python
def get_user_profile(user_id: str) -> UserProfile:
    try:
        return user_service.fetch(user_id)
    except UserNotFoundError:
        # Return default profile instead of breaking
        return UserProfile.default()
    except ServiceUnavailableError:
        # Re-raise with more context
        raise ServiceError(
            "Unable to fetch user profile",
            user_id=user_id,
            retry_after=30
        )
```

### 3. Don't Swallow Errors
Never silently ignore errors.

```python
# ❌ Bad: Error ignored
try:
    process_data()
except Exception:
    pass

# ❌ Bad: Error logged but not handled
try:
    process_data()
except Exception as e:
    logger.error(e)
    # And then what?

# ✅ Good: Error properly handled
try:
    process_data()
except ValidationError as e:
    logger.warning("Invalid data", extra={"error": str(e)})
    return ErrorResponse(code="VALIDATION_ERROR", message=str(e))
except ProcessingError as e:
    logger.error("Processing failed", extra={"error": str(e)})
    raise  # Re-raise for higher-level handler
```

## Exception Hierarchy

```python
# Define domain-specific exceptions
class AppError(Exception):
    """Base exception for application errors."""
    def __init__(self, message: str, code: str = None, **context):
        super().__init__(message)
        self.message = message
        self.code = code or "INTERNAL_ERROR"
        self.context = context

class ValidationError(AppError):
    """Input validation failed."""
    def __init__(self, message: str, field: str = None, **context):
        super().__init__(message, code="VALIDATION_ERROR", field=field, **context)

class NotFoundError(AppError):
    """Resource not found."""
    def __init__(self, resource: str, identifier: str):
        super().__init__(
            f"{resource} not found: {identifier}",
            code="NOT_FOUND",
            resource=resource,
            identifier=identifier
        )

class ConflictError(AppError):
    """Resource conflict (duplicate, state conflict)."""
    pass

class ExternalServiceError(AppError):
    """External service failed."""
    def __init__(self, service: str, message: str, retry_after: int = None):
        super().__init__(
            message,
            code="EXTERNAL_SERVICE_ERROR",
            service=service,
            retry_after=retry_after
        )
```

## HTTP API Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      },
      {
        "field": "age",
        "message": "Must be a positive integer"
      }
    ],
    "request_id": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Error to HTTP Status Mapping

```python
ERROR_STATUS_MAP = {
    ValidationError: 400,      # Bad Request
    AuthenticationError: 401,  # Unauthorized
    AuthorizationError: 403,   # Forbidden
    NotFoundError: 404,        # Not Found
    ConflictError: 409,        # Conflict
    RateLimitError: 429,       # Too Many Requests
    ExternalServiceError: 502, # Bad Gateway
    AppError: 500,             # Internal Server Error
}

@app.exception_handler(AppError)
def handle_app_error(request: Request, exc: AppError):
    status_code = ERROR_STATUS_MAP.get(type(exc), 500)
    return JSONResponse(
        status_code=status_code,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "request_id": request.state.request_id,
                "timestamp": datetime.utcnow().isoformat()
            }
        }
    )
```

## Retry and Circuit Breaker

### Retry with Backoff

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type(TransientError)
)
def call_external_api():
    response = requests.get("https://api.external.com/data")
    if response.status_code == 503:
        raise TransientError("Service temporarily unavailable")
    return response.json()
```

### Circuit Breaker

```python
from circuitbreaker import circuit

@circuit(
    failure_threshold=5,
    recovery_timeout=30,
    expected_exception=ExternalServiceError
)
def call_payment_gateway(amount: float):
    response = payment_api.charge(amount)
    if not response.success:
        raise ExternalServiceError("payment-gateway", response.error)
    return response
```

## Error Logging

```python
def handle_error(error: Exception, context: dict = None):
    """Centralize error handling and logging."""
    
    error_id = str(uuid4())
    
    if isinstance(error, ValidationError):
        # Expected error, log as warning
        logger.warning(
            "Validation error",
            extra={
                "error_id": error_id,
                "error_code": error.code,
                "message": error.message,
                **(context or {}),
                **error.context
            }
        )
    elif isinstance(error, AppError):
        # Application error, log as error
        logger.error(
            "Application error",
            extra={
                "error_id": error_id,
                "error_code": error.code,
                "message": error.message,
                **(context or {}),
                **error.context
            }
        )
    else:
        # Unexpected error, log with full stack trace
        logger.exception(
            "Unexpected error",
            extra={
                "error_id": error_id,
                "error_type": type(error).__name__,
                **(context or {})
            }
        )
    
    return error_id
```

## Input Validation

```python
from pydantic import BaseModel, validator, ValidationError as PydanticError

class CreateUserRequest(BaseModel):
    email: str
    name: str
    age: int
    
    @validator('email')
    def validate_email(cls, v):
        if '@' not in v:
            raise ValueError('Invalid email format')
        return v.lower()
    
    @validator('age')
    def validate_age(cls, v):
        if v < 0 or v > 150:
            raise ValueError('Age must be between 0 and 150')
        return v

def create_user(data: dict):
    try:
        request = CreateUserRequest(**data)
    except PydanticError as e:
        raise ValidationError(
            "Invalid request data",
            details=[
                {"field": err["loc"][0], "message": err["msg"]}
                for err in e.errors()
            ]
        )
    
    return user_service.create(request)
```

## Async/Await Error Handling

```python
import asyncio

async def fetch_with_timeout(url: str, timeout: int = 5):
    try:
        async with asyncio.timeout(timeout):
            return await http_client.get(url)
    except asyncio.TimeoutError:
        raise ExternalServiceError(
            service=url,
            message="Request timed out",
            retry_after=10
        )

async def fetch_all_data(urls: list[str]):
    """Fetch multiple URLs, handling individual failures."""
    results = await asyncio.gather(
        *[fetch_with_timeout(url) for url in urls],
        return_exceptions=True  # Don't fail everything if one fails
    )
    
    successful = []
    failed = []
    
    for url, result in zip(urls, results):
        if isinstance(result, Exception):
            failed.append({"url": url, "error": str(result)})
        else:
            successful.append(result)
    
    return {"data": successful, "errors": failed}
```

## Error Handling Checklist

### Design
- [ ] Exception hierarchy defined
- [ ] Expected vs unexpected errors separated
- [ ] Standardized error response format
- [ ] HTTP status code mapping

### Implementation
- [ ] Input validation at the start
- [ ] Specific try/catch (not generic)
- [ ] Errors are not silenced
- [ ] Sufficient context in exceptions

### Operations
- [ ] Errors logged with context
- [ ] Error IDs for tracking
- [ ] Alerts for critical errors
- [ ] Retry/circuit breaker where appropriate
