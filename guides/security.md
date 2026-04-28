# Security

Comprehensive guide for security practices in software development.

## OWASP Top 10 (2021)

### 1. Broken Access Control

```python
# ❌ Bad: No ownership verification
@app.get("/orders/{order_id}")
def get_order(order_id: str):
    return order_repo.find(order_id)

# ✅ Good: Verify user owns the resource
@app.get("/orders/{order_id}")
def get_order(order_id: str, current_user: User = Depends(get_current_user)):
    order = order_repo.find(order_id)
    if order.user_id != current_user.id:
        raise HTTPException(403, "Access denied")
    return order
```

**Common Vulnerabilities:**
- IDOR (Insecure Direct Object References)
- Missing function-level access control
- CORS misconfiguration
- Path traversal

**Mitigations:**
```python
# Role-based access control
def require_role(allowed_roles: list[str]):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, current_user: User, **kwargs):
            if current_user.role not in allowed_roles:
                raise PermissionDenied("Insufficient permissions")
            return func(*args, current_user=current_user, **kwargs)
        return wrapper
    return decorator

@require_role(["admin", "manager"])
def delete_user(user_id: str, current_user: User):
    pass
```

### 2. Cryptographic Failures

```python
# ❌ Bad: Weak hashing
password_hash = hashlib.md5(password.encode()).hexdigest()
password_hash = hashlib.sha1(password.encode()).hexdigest()

# ✅ Good: Strong hashing with salt
from argon2 import PasswordHasher
ph = PasswordHasher(
    time_cost=3,
    memory_cost=65536,
    parallelism=4
)
password_hash = ph.hash(password)

# Verify password
try:
    ph.verify(stored_hash, provided_password)
except argon2.exceptions.VerifyMismatchError:
    raise InvalidCredentials()
```

**Data Classification:**
| Level | Examples | Encryption |
|-------|----------|------------|
| Public | Marketing content | None |
| Internal | Internal docs | TLS in transit |
| Confidential | Customer data | TLS + encryption at rest |
| Restricted | Passwords, PII, financial | TLS + AES-256 + key rotation |

### 3. Injection

```python
# ❌ Bad: SQL Injection
query = f"SELECT * FROM users WHERE id = {user_id}"
query = f"SELECT * FROM users WHERE name = '{name}'"

# ✅ Good: Parameterized queries
query = "SELECT * FROM users WHERE id = %s"
cursor.execute(query, (user_id,))

# Using ORM (SQLAlchemy)
user = session.query(User).filter(User.id == user_id).first()
```

```javascript
// ❌ Bad: Command injection
const { exec } = require('child_process');
exec(`convert ${userInput}.png output.jpg`);

// ✅ Good: Use safe APIs
const { execFile } = require('child_process');
execFile('convert', [sanitizedInput + '.png', 'output.jpg']);
```

```python
# ❌ Bad: LDAP Injection
query = f"(&(uid={username})(password={password}))"

# ✅ Good: Escape special characters
from ldap3.utils.conv import escape_filter_chars
safe_username = escape_filter_chars(username)
query = f"(&(uid={safe_username}))"
```

### 4. Insecure Design

**Threat Modeling Questions:**
- What assets need protection?
- Who are potential attackers?
- What are attack vectors?
- What's the impact of breach?

**Secure Design Patterns:**
```python
# Defense in depth
class PaymentService:
    def process_payment(self, payment: Payment, user: User):
        # Layer 1: Authentication
        self._verify_authentication(user)
        
        # Layer 2: Authorization
        self._verify_authorization(user, payment)
        
        # Layer 3: Input validation
        self._validate_payment(payment)
        
        # Layer 4: Rate limiting
        self._check_rate_limit(user)
        
        # Layer 5: Fraud detection
        self._check_fraud_signals(payment, user)
        
        # Layer 6: Process with audit
        result = self._execute_payment(payment)
        self._audit_log(payment, user, result)
        
        return result
```

### 5. Security Misconfiguration

```yaml
# Security configuration checklist
Production:
  - [ ] Debug mode disabled
  - [ ] Default credentials changed
  - [ ] Unnecessary features disabled
  - [ ] Error messages don't leak info
  - [ ] Security headers configured
  - [ ] HTTPS enforced
  - [ ] Directory listing disabled

# Example: Secure FastAPI configuration
app = FastAPI(
    debug=False,
    docs_url=None if PRODUCTION else "/docs",
    redoc_url=None if PRODUCTION else "/redoc",
)
```

**Security Headers:**
```python
SECURITY_HEADERS = {
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "X-XSS-Protection": "1; mode=block",
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",
    "Content-Security-Policy": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'",
    "Referrer-Policy": "strict-origin-when-cross-origin",
    "Permissions-Policy": "geolocation=(), microphone=(), camera=()",
    "Cache-Control": "no-store, no-cache, must-revalidate",
}

@app.middleware("http")
async def add_security_headers(request, call_next):
    response = await call_next(request)
    for header, value in SECURITY_HEADERS.items():
        response.headers[header] = value
    return response
```

### 6. Vulnerable and Outdated Components

```bash
# Regular vulnerability scanning
# Python
pip-audit
safety check

# JavaScript
npm audit
yarn audit
npx snyk test

# Container images
trivy image myapp:latest
docker scan myapp:latest

# Infrastructure
tfsec .
checkov -d .
```

**Dependency Management:**
```json
// package.json - Pin versions and audit
{
  "scripts": {
    "audit": "npm audit --production",
    "audit:fix": "npm audit fix",
    "outdated": "npm outdated"
  },
  "overrides": {
    "vulnerable-package": "^2.0.0"
  }
}
```

### 7. Identification and Authentication Failures

```python
# Secure authentication configuration
AUTH_CONFIG = {
    "password_min_length": 12,
    "password_require_uppercase": True,
    "password_require_lowercase": True,
    "password_require_digit": True,
    "password_require_special": True,
    "max_login_attempts": 5,
    "lockout_duration_minutes": 15,
    "session_timeout_minutes": 30,
    "mfa_required_for": ["admin", "financial"],
    "password_history_count": 12,
    "jwt_expiry_minutes": 15,
    "refresh_token_expiry_days": 7,
}

class AuthenticationService:
    def authenticate(self, credentials: Credentials) -> AuthResult:
        # Check lockout
        if self._is_locked_out(credentials.email):
            raise AccountLockedError(
                "Account locked. Try again in 15 minutes."
            )
        
        user = self._get_user(credentials.email)
        if not user or not self._verify_password(credentials.password, user.password_hash):
            self._record_failed_attempt(credentials.email)
            raise InvalidCredentialsError()
        
        # Clear failed attempts on success
        self._clear_failed_attempts(credentials.email)
        
        # Check if MFA required
        if user.role in AUTH_CONFIG["mfa_required_for"]:
            return AuthResult(requires_mfa=True, mfa_token=self._generate_mfa_token(user))
        
        return self._create_session(user)
```

### 8. Software and Data Integrity Failures

```python
# Verify webhook signatures
import hmac
import hashlib

def verify_webhook_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

@app.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    payload = await request.body()
    signature = request.headers.get("Stripe-Signature")
    
    if not verify_webhook_signature(payload, signature, STRIPE_WEBHOOK_SECRET):
        raise HTTPException(400, "Invalid signature")
    
    # Process webhook...
```

**Supply Chain Security:**
```yaml
# .github/workflows/security.yml
- name: Verify commit signatures
  run: |
    git verify-commit HEAD
    
- name: Check dependency checksums
  run: |
    npm ci --ignore-scripts
    pip install --require-hashes -r requirements.txt
    
- name: SBOM generation
  uses: anchore/sbom-action@v0
  with:
    artifact-name: sbom.spdx
```

### 9. Security Logging and Monitoring Failures

```python
import structlog
from datetime import datetime

logger = structlog.get_logger()

# Security events to log
SECURITY_EVENTS = [
    "login_success",
    "login_failure",
    "logout",
    "password_change",
    "mfa_enabled",
    "mfa_disabled",
    "permission_denied",
    "suspicious_activity",
    "api_key_created",
    "api_key_revoked",
]

def log_security_event(
    event_type: str,
    user_id: str = None,
    ip_address: str = None,
    user_agent: str = None,
    details: dict = None
):
    logger.info(
        "security_event",
        event_type=event_type,
        user_id=user_id,
        ip_address=ip_address,
        user_agent=user_agent,
        timestamp=datetime.utcnow().isoformat(),
        details=details or {}
    )

# Example usage
log_security_event(
    event_type="login_failure",
    user_id=None,
    ip_address=request.client.host,
    user_agent=request.headers.get("user-agent"),
    details={"email": email, "reason": "invalid_password"}
)
```

### 10. Server-Side Request Forgery (SSRF)

```python
from urllib.parse import urlparse
import ipaddress

# ❌ Bad: Arbitrary URL fetch
def fetch_url(url: str):
    return requests.get(url)

# ✅ Good: URL validation and allowlist
ALLOWED_HOSTS = ["api.trusted.com", "cdn.trusted.com"]
BLOCKED_IP_RANGES = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("127.0.0.0/8"),
    ipaddress.ip_network("169.254.0.0/16"),
]

def is_safe_url(url: str) -> bool:
    try:
        parsed = urlparse(url)
        
        # Check scheme
        if parsed.scheme not in ["http", "https"]:
            return False
        
        # Check host allowlist
        if parsed.hostname not in ALLOWED_HOSTS:
            return False
        
        # Resolve and check IP
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
        for blocked_range in BLOCKED_IP_RANGES:
            if ip in blocked_range:
                return False
        
        return True
    except Exception:
        return False

def fetch_url(url: str):
    if not is_safe_url(url):
        raise ValueError("URL not allowed")
    return requests.get(url, timeout=10, allow_redirects=False)
```

---

## Authentication and Authorization

### JWT Best Practices

```python
from datetime import datetime, timedelta
from jose import jwt, JWTError
import secrets

# Secure JWT configuration
JWT_CONFIG = {
    "algorithm": "RS256",  # Asymmetric preferred
    "access_token_expire_minutes": 15,
    "refresh_token_expire_days": 7,
    "issuer": "https://api.yourapp.com",
    "audience": "yourapp-api",
}

def create_access_token(user_id: str, roles: list[str]) -> str:
    now = datetime.utcnow()
    payload = {
        "sub": user_id,
        "roles": roles,
        "iat": now,
        "exp": now + timedelta(minutes=JWT_CONFIG["access_token_expire_minutes"]),
        "iss": JWT_CONFIG["issuer"],
        "aud": JWT_CONFIG["audience"],
        "jti": secrets.token_urlsafe(32),  # Unique ID for revocation
    }
    return jwt.encode(payload, PRIVATE_KEY, algorithm=JWT_CONFIG["algorithm"])

def verify_token(token: str) -> dict:
    try:
        payload = jwt.decode(
            token,
            PUBLIC_KEY,
            algorithms=[JWT_CONFIG["algorithm"]],
            issuer=JWT_CONFIG["issuer"],
            audience=JWT_CONFIG["audience"],
        )
        
        # Check if token is revoked
        if is_token_revoked(payload["jti"]):
            raise JWTError("Token revoked")
        
        return payload
    except JWTError as e:
        raise InvalidTokenError(str(e))
```

### OAuth 2.0 / OIDC

```python
# OAuth 2.0 state parameter to prevent CSRF
def generate_oauth_state() -> str:
    state = secrets.token_urlsafe(32)
    redis.setex(f"oauth:state:{state}", 600, "valid")  # 10 min expiry
    return state

def verify_oauth_state(state: str) -> bool:
    key = f"oauth:state:{state}"
    if redis.get(key):
        redis.delete(key)  # One-time use
        return True
    return False

# PKCE for public clients
def generate_pkce_verifier() -> str:
    return secrets.token_urlsafe(32)

def generate_pkce_challenge(verifier: str) -> str:
    digest = hashlib.sha256(verifier.encode()).digest()
    return base64.urlsafe_b64encode(digest).decode().rstrip("=")
```

### API Key Management

```python
import hashlib
import secrets

class APIKeyService:
    def create_api_key(self, user_id: str, name: str) -> tuple[str, str]:
        # Generate key (shown once to user)
        raw_key = f"sk_{secrets.token_urlsafe(32)}"
        
        # Store hash (never store raw key)
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        
        # Store prefix for identification
        key_prefix = raw_key[:8]
        
        api_key = APIKey(
            id=str(uuid4()),
            user_id=user_id,
            name=name,
            key_hash=key_hash,
            key_prefix=key_prefix,
            created_at=datetime.utcnow(),
            last_used_at=None,
        )
        self.repository.save(api_key)
        
        return api_key.id, raw_key  # Return raw key only once

    def verify_api_key(self, raw_key: str) -> APIKey:
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        api_key = self.repository.find_by_hash(key_hash)
        
        if not api_key:
            raise InvalidAPIKeyError()
        
        if api_key.revoked_at:
            raise RevokedAPIKeyError()
        
        # Update last used
        api_key.last_used_at = datetime.utcnow()
        self.repository.save(api_key)
        
        return api_key
```

---

## Data Protection

### Encryption at Rest

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

# Symmetric encryption for data at rest
class EncryptionService:
    def __init__(self, key: bytes):
        self.fernet = Fernet(key)
    
    def encrypt(self, data: str) -> str:
        return self.fernet.encrypt(data.encode()).decode()
    
    def decrypt(self, encrypted: str) -> str:
        return self.fernet.decrypt(encrypted.encode()).decode()

# AES-GCM for field-level encryption
class FieldEncryption:
    def __init__(self, key: bytes):
        self.key = key
    
    def encrypt(self, plaintext: str) -> bytes:
        nonce = os.urandom(12)
        aesgcm = AESGCM(self.key)
        ciphertext = aesgcm.encrypt(nonce, plaintext.encode(), None)
        return nonce + ciphertext
    
    def decrypt(self, data: bytes) -> str:
        nonce = data[:12]
        ciphertext = data[12:]
        aesgcm = AESGCM(self.key)
        return aesgcm.decrypt(nonce, ciphertext, None).decode()
```

### PII Handling

```python
# PII field classification
PII_FIELDS = {
    "ssn": "restricted",
    "tax_id": "restricted",
    "credit_card": "restricted",
    "password": "restricted",
    "email": "confidential",
    "phone": "confidential",
    "address": "confidential",
    "name": "internal",
    "birth_date": "confidential",
}

class PIIHandler:
    def __init__(self, encryption_service: EncryptionService):
        self.encryption = encryption_service
    
    def mask_for_display(self, field: str, value: str) -> str:
        """Mask PII for display (e.g., in logs, UI)."""
        if field == "email":
            local, domain = value.split("@")
            return f"{local[0]}***@{domain}"
        elif field == "phone":
            return f"***-***-{value[-4:]}"
        elif field == "credit_card":
            return f"****-****-****-{value[-4:]}"
        elif field == "ssn":
            return f"***-**-{value[-4:]}"
        else:
            return "***REDACTED***"
    
    def sanitize_for_logging(self, data: dict) -> dict:
        """Remove or mask PII before logging."""
        sanitized = {}
        for key, value in data.items():
            if key in PII_FIELDS:
                if PII_FIELDS[key] == "restricted":
                    sanitized[key] = "[REDACTED]"
                else:
                    sanitized[key] = self.mask_for_display(key, str(value))
            else:
                sanitized[key] = value
        return sanitized
```

### Data Retention and Deletion

```python
class DataRetentionService:
    RETENTION_POLICIES = {
        "user_data": timedelta(days=365 * 7),  # 7 years
        "logs": timedelta(days=90),
        "sessions": timedelta(days=30),
        "audit_trails": timedelta(days=365 * 10),  # 10 years
    }
    
    def schedule_deletion(self, data_type: str, record_id: str):
        retention = self.RETENTION_POLICIES.get(data_type)
        if retention:
            deletion_date = datetime.utcnow() + retention
            self.deletion_queue.schedule(
                record_id=record_id,
                data_type=data_type,
                scheduled_for=deletion_date
            )
    
    def handle_deletion_request(self, user_id: str):
        """GDPR right to be forgotten."""
        # 1. Mark user for deletion
        user = self.user_repo.find(user_id)
        user.deletion_requested_at = datetime.utcnow()
        
        # 2. Anonymize data that must be retained
        self._anonymize_audit_logs(user_id)
        
        # 3. Delete data that can be deleted
        self._delete_user_content(user_id)
        
        # 4. Notify third parties
        self._notify_data_processors(user_id)
        
        # 5. Complete deletion
        self.user_repo.delete(user_id)
```

---

## Secrets Management

### Environment Variables

```python
# ❌ Bad: Hardcoded secrets
API_KEY = "sk-1234567890abcdef"
DATABASE_URL = "postgres://user:password@localhost/db"

# ✅ Good: Environment variables
import os

API_KEY = os.environ["API_KEY"]
DATABASE_URL = os.environ["DATABASE_URL"]

# With validation
def get_required_env(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        raise EnvironmentError(f"Required environment variable {name} not set")
    return value
```

### Secrets Manager Integration

```python
# AWS Secrets Manager
import boto3
import json

class SecretsManager:
    def __init__(self):
        self.client = boto3.client("secretsmanager")
        self._cache = {}
    
    def get_secret(self, secret_name: str) -> dict:
        if secret_name not in self._cache:
            response = self.client.get_secret_value(SecretId=secret_name)
            self._cache[secret_name] = json.loads(response["SecretString"])
        return self._cache[secret_name]

# HashiCorp Vault
import hvac

class VaultClient:
    def __init__(self, url: str, token: str):
        self.client = hvac.Client(url=url, token=token)
    
    def get_secret(self, path: str) -> dict:
        response = self.client.secrets.kv.v2.read_secret_version(path=path)
        return response["data"]["data"]
```

### Secret Rotation

```python
class SecretRotation:
    def rotate_database_credentials(self):
        # 1. Generate new credentials
        new_password = secrets.token_urlsafe(32)
        
        # 2. Create new user in database
        self._create_db_user(new_password)
        
        # 3. Update secret in secrets manager
        self.secrets_manager.update_secret(
            "database-credentials",
            {"password": new_password}
        )
        
        # 4. Wait for applications to pick up new credentials
        time.sleep(60)
        
        # 5. Revoke old credentials
        self._revoke_old_db_user()
```

---

## Security Scanning

### Static Application Security Testing (SAST)

```yaml
# GitHub Actions SAST workflow
name: Security Scan

on: [push, pull_request]

jobs:
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
      
      - name: Run Bandit (Python)
        run: |
          pip install bandit
          bandit -r src/ -f json -o bandit-report.json
      
      - name: Run ESLint Security
        run: |
          npm install eslint-plugin-security
          npx eslint --ext .js,.ts src/ --format json -o eslint-report.json
```

### Dynamic Application Security Testing (DAST)

```yaml
# OWASP ZAP scan
- name: OWASP ZAP Scan
  uses: zaproxy/action-full-scan@v0.4.0
  with:
    target: "https://staging.yourapp.com"
    rules_file_name: ".zap/rules.tsv"
    cmd_options: "-a"
```

### Dependency Scanning

```yaml
# Continuous dependency scanning
- name: Dependency Review
  uses: actions/dependency-review-action@v3
  with:
    fail-on-severity: high
    
- name: Snyk Security Scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

### Container Scanning

```yaml
# Trivy container scan
- name: Container Scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: "myapp:${{ github.sha }}"
    format: "sarif"
    output: "trivy-results.sarif"
    severity: "CRITICAL,HIGH"
```

### Secret Scanning

```yaml
# Detect secrets in code
- name: Gitleaks
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

# Pre-commit hook
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ["--baseline", ".secrets.baseline"]
```

---

## Security Checklist

### Development Phase

- [ ] Input validation on all user inputs
- [ ] Output encoding to prevent XSS
- [ ] Parameterized queries (no SQL injection)
- [ ] Authentication on all sensitive endpoints
- [ ] Authorization checks on server-side
- [ ] CSRF protection on state-changing operations
- [ ] Secure session management
- [ ] Error messages don't leak sensitive info

### Code Review

- [ ] No hardcoded secrets
- [ ] No sensitive data in logs
- [ ] Cryptography uses standard libraries
- [ ] Rate limiting implemented
- [ ] Input size limits enforced
- [ ] File upload restrictions in place

### Deployment

- [ ] HTTPS enforced everywhere
- [ ] Security headers configured
- [ ] CORS properly configured
- [ ] Secrets in secret manager
- [ ] Dependencies up to date
- [ ] Container images scanned
- [ ] Least privilege IAM roles

### Monitoring

- [ ] Security events logged
- [ ] Failed login alerts
- [ ] Anomaly detection active
- [ ] Incident response plan ready
- [ ] Regular penetration testing
- [ ] Vulnerability disclosure program
