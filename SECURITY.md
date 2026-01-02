# Security & Compliance

## Security Architecture

### Defense in Depth Strategy

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: Network Security                              │
│  - AWS Shield (DDoS protection)                        │
│  - AWS WAF (Web Application Firewall)                  │
│  - CloudFront (CDN with bot protection)                │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  Layer 2: API Gateway Security                         │
│  - Rate limiting (per-user, per-IP)                    │
│  - Request validation                                  │
│  - JWT token verification                              │
│  - API key authentication (Enterprise)                 │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  Layer 3: Application Security                         │
│  - Input validation (Pydantic)                         │
│  - SQL injection prevention (parameterized queries)    │
│  - XSS prevention (content security policy)            │
│  - CSRF protection                                     │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  Layer 4: Data Security                                │
│  - Encryption at rest (AES-256)                        │
│  - Encryption in transit (TLS 1.3)                     │
│  - Secrets management (AWS Secrets Manager)            │
│  - Database encryption (RDS encryption)                │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  Layer 5: Access Control                               │
│  - RBAC (Role-Based Access Control)                    │
│  - IAM policies (AWS)                                  │
│  - Kubernetes RBAC                                     │
│  - Principle of least privilege                        │
└─────────────────────────────────────────────────────────┘
```

---

## Authentication & Authorization

### OAuth 2.0 Flow (Google Sign-In)

```
┌──────────┐                                              ┌───────────┐
│          │                                              │           │
│  Client  │                                              │  Google   │
│  (Web)   │                                              │  OAuth    │
│          │                                              │           │
└────┬─────┘                                              └─────┬─────┘
     │                                                          │
     │ 1. Initiate Google Sign-In                              │
     ├─────────────────────────────────────────────────────────►
     │                                                          │
     │ 2. User authenticates with Google                       │
     │                                                          │
     │ 3. Google returns ID token                              │
     ◄─────────────────────────────────────────────────────────┤
     │                                                          │
┌────▼─────┐                                              ┌─────┴─────┐
│          │ 4. Send ID token to backend                  │           │
│  Client  ├──────────────────────────────────────────────►   Auth    │
│          │                                              │  Service  │
│          │ 5. Verify token with Google                  │           │
│          │                  ┌──────────────────────────►│           │
│          │                  │ 6. Token valid            │           │
│          │                  └───────────────────────────┤           │
│          │                                              │           │
│          │ 7. Create/update user in DB                  │           │
│          │                  ┌──────────────────────────►│           │
│          │                  │ 8. User record            │           │
│          │                  └───────────────────────────┤           │
│          │                                              │           │
│          │ 9. Generate JWT access token                 │           │
│          │◄──────────────────────────────────────────────┤           │
└──────────┘                                              └───────────┘
```

### JWT Token Structure

```python
# JWT Payload
{
    "sub": "user_uuid",              # Subject (user ID)
    "email": "user@example.com",
    "name": "John Doe",
    "role": "enterprise",            # free, enterprise, admin
    "iat": 1704067200,              # Issued at timestamp
    "exp": 1704153600,              # Expiration timestamp (24 hours)
    "jti": "unique_token_id"        # JWT ID (for revocation)
}

# Token Generation
import jwt
from datetime import datetime, timedelta

def create_access_token(user: User) -> str:
    payload = {
        "sub": str(user.id),
        "email": user.email,
        "name": user.name,
        "role": user.role,
        "iat": datetime.utcnow(),
        "exp": datetime.utcnow() + timedelta(hours=24),
        "jti": str(uuid.uuid4())
    }
    
    return jwt.encode(
        payload,
        settings.JWT_SECRET_KEY,
        algorithm="HS256"
    )

# Token Verification
async def verify_access_token(token: str) -> User:
    try:
        payload = jwt.decode(
            token,
            settings.JWT_SECRET_KEY,
            algorithms=["HS256"]
        )
        
        # Check if token is revoked (blacklist check)
        if await is_token_revoked(payload["jti"]):
            raise HTTPException(status_code=401, detail="Token revoked")
        
        # Load user from database
        user = await db.get_user_by_id(payload["sub"])
        if not user:
            raise HTTPException(status_code=401, detail="User not found")
        
        return user
    
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

### API Key Authentication (Enterprise)

```python
# API Key Generation
import secrets
import hashlib

def generate_api_key() -> tuple[str, str]:
    """
    Generate API key and its hash
    
    Returns:
        (api_key, key_hash) - Store hash in DB, return key to user once
    """
    # Generate 32-byte random key
    api_key = f"sk_live_{secrets.token_urlsafe(32)}"
    
    # Hash for storage (SHA-256)
    key_hash = hashlib.sha256(api_key.encode()).hexdigest()
    
    return api_key, key_hash

# API Key Validation Middleware
from fastapi import Security, HTTPException
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def verify_api_key(api_key: str = Security(api_key_header)) -> User:
    if not api_key:
        raise HTTPException(status_code=401, detail="API key required")
    
    # Hash incoming key
    key_hash = hashlib.sha256(api_key.encode()).hexdigest()
    
    # Look up in database
    api_key_record = await db.get_api_key_by_hash(key_hash)
    
    if not api_key_record or not api_key_record.is_active:
        raise HTTPException(status_code=401, detail="Invalid API key")
    
    # Check expiration
    if api_key_record.expires_at and api_key_record.expires_at < datetime.utcnow():
        raise HTTPException(status_code=401, detail="API key expired")
    
    # Update last used timestamp
    await db.update_api_key_last_used(api_key_record.id)
    
    # Load user
    user = await db.get_user_by_id(api_key_record.user_id)
    
    return user

# Protected endpoint
@app.get("/api/v1/market-data/{symbol}")
async def get_market_data(
    symbol: str,
    user: User = Depends(verify_api_key)
):
    # Check rate limit
    if not await check_rate_limit(user.id):
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    
    return await fetch_market_data(symbol)
```

---

## Rate Limiting

### Multi-Layer Rate Limiting

```python
# rate_limiter.py
from fastapi import Request, HTTPException
from datetime import datetime, timedelta
import redis

redis_client = redis.Redis(host='dragonfly.svc.cluster.local', port=6379)

class RateLimiter:
    """Multi-tier rate limiting"""
    
    @staticmethod
    async def check_ip_rate_limit(request: Request) -> bool:
        """
        IP-based rate limiting (DDoS protection)
        Limit: 100 requests per minute per IP
        """
        ip = request.client.host
        key = f"ratelimit:ip:{ip}:minute"
        
        current = redis_client.incr(key)
        if current == 1:
            redis_client.expire(key, 60)
        
        if current > 100:
            raise HTTPException(
                status_code=429,
                detail="Too many requests from this IP. Try again later."
            )
        
        return True
    
    @staticmethod
    async def check_user_rate_limit(user_id: str, tier: str) -> bool:
        """
        User-based rate limiting (per tier)
        
        Free tier: 60 requests/minute, 1000 requests/day
        Enterprise: 1000 requests/minute, unlimited daily
        """
        limits = {
            'free': {'minute': 60, 'day': 1000},
            'enterprise': {'minute': 1000, 'day': 999999}
        }
        
        user_limits = limits.get(tier, limits['free'])
        
        # Check minute limit
        minute_key = f"ratelimit:user:{user_id}:minute"
        minute_count = redis_client.incr(minute_key)
        if minute_count == 1:
            redis_client.expire(minute_key, 60)
        
        if minute_count > user_limits['minute']:
            raise HTTPException(
                status_code=429,
                detail=f"Rate limit exceeded: {user_limits['minute']} requests/minute"
            )
        
        # Check daily limit
        day_key = f"ratelimit:user:{user_id}:day"
        day_count = redis_client.incr(day_key)
        if day_count == 1:
            redis_client.expire(day_key, 86400)  # 24 hours
        
        if day_count > user_limits['day']:
            raise HTTPException(
                status_code=429,
                detail=f"Daily limit exceeded: {user_limits['day']} requests/day"
            )
        
        return True
    
    @staticmethod
    async def check_endpoint_rate_limit(user_id: str, endpoint: str, tier: str) -> bool:
        """
        Endpoint-specific rate limiting
        
        Expensive operations (backtest, ML inference) have lower limits
        """
        endpoint_limits = {
            'backtest': {'free': 3, 'enterprise': 100},  # per day
            'ml_predict': {'free': 10, 'enterprise': 1000}  # per hour
        }
        
        if endpoint not in endpoint_limits:
            return True  # No specific limit
        
        limits = endpoint_limits[endpoint]
        limit = limits.get(tier, limits['free'])
        
        key = f"ratelimit:user:{user_id}:endpoint:{endpoint}"
        current = redis_client.incr(key)
        
        if current == 1:
            # Set expiry based on endpoint type
            ttl = 86400 if endpoint == 'backtest' else 3600
            redis_client.expire(key, ttl)
        
        if current > limit:
            raise HTTPException(
                status_code=429,
                detail=f"{endpoint} limit exceeded: {limit} requests allowed"
            )
        
        return True

# Middleware
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    # IP-based rate limiting (always applied)
    await RateLimiter.check_ip_rate_limit(request)
    
    response = await call_next(request)
    return response
```

---

## Input Validation & Sanitization

### Pydantic Models (Automatic Validation)

```python
from pydantic import BaseModel, Field, validator, EmailStr
from typing import List, Optional
from datetime import datetime

class CreateWatchlistRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    symbols: List[str] = Field(..., min_items=1, max_items=50)
    
    @validator('name')
    def validate_name(cls, v):
        # Prevent XSS
        if any(char in v for char in ['<', '>', '"', "'"]):
            raise ValueError('Name contains invalid characters')
        return v.strip()
    
    @validator('symbols')
    def validate_symbols(cls, v):
        # Ensure valid symbol format
        for symbol in v:
            if not symbol.isalnum() or len(symbol) > 20:
                raise ValueError(f'Invalid symbol: {symbol}')
        return [s.upper() for s in v]

class BacktestRequest(BaseModel):
    strategy_name: str = Field(..., regex=r'^[a-z_]+$')
    symbol: str = Field(..., regex=r'^[A-Z0-9]+$')
    interval: str = Field(..., regex=r'^(1m|5m|15m|1h|1d)$')
    start_date: datetime
    end_date: datetime
    parameters: dict = Field(default_factory=dict)
    
    @validator('end_date')
    def validate_date_range(cls, v, values):
        if 'start_date' in values and v <= values['start_date']:
            raise ValueError('end_date must be after start_date')
        
        # Limit backtest duration
        if 'start_date' in values:
            duration = (v - values['start_date']).days
            if duration > 365 * 3:  # Max 3 years
                raise ValueError('Backtest duration limited to 3 years')
        
        return v
    
    @validator('parameters')
    def validate_parameters(cls, v):
        # Limit nested depth to prevent DoS
        if isinstance(v, dict) and len(str(v)) > 10000:
            raise ValueError('Parameters too large')
        return v

# Usage in endpoint
@app.post("/watchlists")
async def create_watchlist(
    request: CreateWatchlistRequest,  # Automatic validation
    user: User = Depends(get_current_user)
):
    # request.symbols is already validated and sanitized
    watchlist = await db.create_watchlist(
        user.id,
        request.name,
        request.symbols
    )
    return watchlist
```

### SQL Injection Prevention

```python
# WRONG: String concatenation (vulnerable)
def get_user_bad(email: str):
    query = f"SELECT * FROM users WHERE email = '{email}'"
    return db.execute(query)

# Attacker input: ' OR '1'='1' --
# Resulting query: SELECT * FROM users WHERE email = '' OR '1'='1' --'
# Returns all users!

# CORRECT: Parameterized queries
async def get_user_safe(email: str):
    query = "SELECT * FROM users WHERE email = :email"
    return await db.fetch_one(query, values={'email': email})

# Using SQLAlchemy (even safer)
from sqlalchemy import select
from models import User

async def get_user_sqlalchemy(email: str):
    stmt = select(User).where(User.email == email)
    result = await session.execute(stmt)
    return result.scalar_one_or_none()
```

---

## Encryption

### Encryption at Rest

```yaml
# RDS PostgreSQL Encryption
aws rds create-db-instance \
  --db-instance-identifier trading-platform-prod \
  --db-instance-class db.r6i.2xlarge \
  --engine postgres \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/your-key-id

# EBS Volume Encryption (for Kafka, ClickHouse)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Gi
  storageClassName: encrypted-gp3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/your-key-id

# S3 Bucket Encryption
aws s3api put-bucket-encryption \
  --bucket trading-platform-backtest-reports \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789:key/your-key-id"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

### Encryption in Transit

```yaml
# API Gateway - Enforce HTTPS only
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789:certificate/your-cert-id
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: api.tradingplatform.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 80

# PostgreSQL - Enforce SSL connections
# In PostgreSQL config:
ssl = on
ssl_cert_file = '/path/to/server.crt'
ssl_key_file = '/path/to/server.key'
ssl_ca_file = '/path/to/root.crt'

# In application connection string:
DATABASE_URL = "postgresql://user:pass@host:5432/db?sslmode=require"
```

### Secrets Management

```python
# AWS Secrets Manager integration
import boto3
from functools import lru_cache

class SecretsManager:
    def __init__(self):
        self.client = boto3.client('secretsmanager', region_name='us-east-1')
    
    @lru_cache(maxsize=100)
    def get_secret(self, secret_name: str) -> dict:
        """
        Retrieve secret from AWS Secrets Manager
        Cached to reduce API calls
        """
        try:
            response = self.client.get_secret_value(SecretId=secret_name)
            return json.loads(response['SecretString'])
        except Exception as e:
            logger.error(f"Failed to retrieve secret {secret_name}: {e}")
            raise

secrets_manager = SecretsManager()

# Usage
upstox_credentials = secrets_manager.get_secret('prod/upstox/api')
UPSTOX_API_KEY = upstox_credentials['api_key']
UPSTOX_API_SECRET = upstox_credentials['api_secret']

database_credentials = secrets_manager.get_secret('prod/database/postgres')
DATABASE_URL = f"postgresql://{database_credentials['username']}:{database_credentials['password']}@{database_credentials['host']}:5432/{database_credentials['database']}"
```

---

## RBAC (Role-Based Access Control)

### Kubernetes RBAC

```yaml
# Service Account for market-data-service
apiVersion: v1
kind: ServiceAccount
metadata:
  name: market-data-service
  namespace: trading-platform

---
# Role: Read-only access to ConfigMaps and Secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: market-data-service-role
  namespace: trading-platform
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]

---
# RoleBinding: Bind role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: market-data-service-binding
  namespace: trading-platform
subjects:
- kind: ServiceAccount
  name: market-data-service
roleRef:
  kind: Role
  name: market-data-service-role
  apiGroup: rbac.authorization.k8s.io

---
# Use in Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: market-data-service
spec:
  template:
    spec:
      serviceAccountName: market-data-service  # Use restricted service account
      containers:
      - name: market-data
        image: market-data-service:latest
```

---

## Compliance & Auditing

### SEBI Compliance (India)

```python
# Audit logging for all trading-related activities
class AuditLogger:
    """
    Comprehensive audit logging for regulatory compliance
    """
    
    @staticmethod
    async def log_user_action(
        user_id: str,
        action: str,
        resource: str,
        details: dict,
        ip_address: str,
        user_agent: str
    ):
        """
        Log user action for audit trail
        
        Required for SEBI compliance:
        - Who did what
        - When it happened
        - From where (IP)
        - Result (success/failure)
        """
        audit_record = {
            'timestamp': datetime.utcnow().isoformat(),
            'user_id': user_id,
            'action': action,  # 'create_watchlist', 'run_backtest', 'view_signals'
            'resource': resource,
            'details': details,
            'ip_address': ip_address,
            'user_agent': user_agent,
            'status': 'success'  # or 'failure'
        }
        
        # Write to database (immutable log)
        await db.insert_audit_log(audit_record)
        
        # Also send to centralized logging
        logger.info("Audit event", extra=audit_record)

# Usage in endpoints
@app.post("/backtests")
async def run_backtest(
    request: BacktestRequest,
    http_request: Request,
    user: User = Depends(get_current_user)
):
    # Log audit event
    await AuditLogger.log_user_action(
        user_id=str(user.id),
        action='run_backtest',
        resource=f'backtest:{request.strategy_name}:{request.symbol}',
        details={
            'strategy': request.strategy_name,
            'symbol': request.symbol,
            'start_date': request.start_date.isoformat(),
            'end_date': request.end_date.isoformat()
        },
        ip_address=http_request.client.host,
        user_agent=http_request.headers.get('user-agent')
    )
    
    # Execute backtest
    result = await backtest_engine.run(request)
    
    return result
```

### Data Retention Policies

```sql
-- Audit logs: Retain for 7 years (regulatory requirement)
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL,
    user_id UUID,
    action VARCHAR(100),
    resource VARCHAR(255),
    details JSONB,
    ip_address INET,
    status VARCHAR(50),
    
    -- Prevent modification (append-only)
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Create immutable audit log (PostgreSQL 12+)
ALTER TABLE audit_logs SET (
    autovacuum_enabled = false  -- Prevent deletion
);

-- Revoke DELETE and UPDATE permissions
REVOKE DELETE, UPDATE ON audit_logs FROM ALL;

-- Only INSERT allowed
GRANT INSERT ON audit_logs TO application_user;

-- Archive old audit logs to S3 (after 1 year)
-- Keep recent logs in database for fast access
SELECT * FROM audit_logs
WHERE created_at < NOW() - INTERVAL '1 year'
INTO OUTFILE 's3://audit-logs-archive/2023/audit_logs.parquet';
```

---

## Security Monitoring & Incident Response

### Security Alerts

```yaml
# Prometheus alerts for security events
groups:
  - name: security_alerts
    rules:
      # Brute force attack detection
      - alert: BruteForceAttack
        expr: |
          sum(rate(http_requests_total{status="401", endpoint="/auth/login"}[5m])) by (source_ip) > 10
        for: 5m
        labels:
          severity: critical
          team: security
        annotations:
          summary: "Potential brute force attack from {{ $labels.source_ip }}"
      
      # Unusual API access pattern
      - alert: UnusualAPIAccessPattern
        expr: |
          rate(http_requests_total{user_id=~".+"}[5m]) > 100
        labels:
          severity: warning
          team: security
        annotations:
          summary: "Unusual API access rate from user {{ $labels.user_id }}"
      
      # Unauthorized access attempts
      - alert: UnauthorizedAccessAttempts
        expr: |
          sum(rate(http_requests_total{status=~"401|403"}[5m])) > 50
        for: 5m
        labels:
          severity: warning
          team: security
        annotations:
          summary: "High rate of unauthorized access attempts"
```

### Incident Response Plan

```markdown
# Security Incident Response Plan

## Phase 1: Detection & Assessment (0-15 minutes)
1. Alert received (automated monitoring)
2. On-call engineer acknowledges
3. Assess severity:
   - Critical: Data breach, unauthorized access to production
   - High: DDoS attack, service disruption
   - Medium: Failed login attempts spike
   - Low: Single user account compromise

## Phase 2: Containment (15-60 minutes)
**Critical/High Incidents:**
1. Isolate affected systems
2. Revoke compromised credentials
3. Enable IP blocking (WAF rules)
4. Snapshot affected systems for forensics

**Medium Incidents:**
1. Rate limit suspicious IPs
2. Force password reset for affected users
3. Review audit logs

## Phase 3: Eradication (1-4 hours)
1. Identify root cause
2. Patch vulnerabilities
3. Remove malware/backdoors
4. Update firewall rules

## Phase 4: Recovery (4-24 hours)
1. Restore services from clean backups
2. Monitor for re-infection
3. Gradual traffic ramp-up
4. Verify all systems operational

## Phase 5: Post-Incident (24-72 hours)
1. Write incident report
2. Update runbooks
3. Conduct retrospective
4. Implement preventive measures
5. Notify affected users (if required)
6. File regulatory reports (if required)

## Contact Information
- Security Team Lead: security@tradingplatform.com
- AWS Support: (Enterprise Support)
- Legal Counsel: legal@tradingplatform.com
- SEBI Reporting: compliance@tradingplatform.com
```

---

This comprehensive security documentation ensures the platform meets institutional-grade security standards and regulatory compliance requirements.
