# User Story 02.1: JWT-Based Authentication

## Story
As a **system user**, I want to **authenticate securely using JWT tokens** so that **I can access the certificate management system with confidence**.

## Acceptance Criteria
```gherkin
Feature: JWT-Based Authentication
  As a system user
  I want to authenticate using secure JWT tokens
  So that my access to the system is properly controlled

  Scenario: User login with credentials
    Given I have valid username and password
    When I submit my credentials to the login endpoint
    Then I should receive a JWT access token
    And I should receive a refresh token
    And the tokens should be signed with RS256
    And the response time should be under 200ms

  Scenario: Token-based API access
    Given I have a valid JWT access token
    When I make an API request with the token
    Then the request should be authorised
    And my user context should be available
    And the token should contain my roles and permissions

  Scenario: Token refresh
    Given my access token is about to expire
    When I use my refresh token to get a new access token
    Then I should receive a new access token
    And the refresh token should remain valid
    And there should be no service interruption

  Scenario: Invalid authentication attempts
    Given I provide invalid credentials
    When I attempt to login
    Then I should receive an authentication error
    And the error should not reveal whether username or password was wrong
    And failed attempts should be rate-limited
    And multiple failures should trigger account protection
```

## Technical Details

### JWT Implementation
```python
from datetime import datetime, timedelta
from typing import Optional, Dict, List
import jwt
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa
from fastapi import HTTPException, status, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import redis
from argon2 import PasswordHasher
from pydantic import BaseModel
import secrets

class TokenPayload(BaseModel):
    sub: str  # Subject (user ID)
    exp: int  # Expiration time
    iat: int  # Issued at
    jti: str  # JWT ID for revocation
    roles: List[str]
    permissions: List[str]
    organisation_id: str

class AuthenticationService:
    def __init__(self, config: Dict):
        self.config = config
        self.redis_client = redis.Redis(
            host=config['redis_host'],
            port=config['redis_port'],
            decode_responses=True
        )
        self.password_hasher = PasswordHasher()
        self.security = HTTPBearer()
        self._load_keys()
    
    def _load_keys(self):
        """Load RSA keys for JWT signing"""
        # Load or generate RSA keys
        try:
            with open(self.config['private_key_path'], 'rb') as f:
                self.private_key = serialization.load_pem_private_key(
                    f.read(),
                    password=self.config['key_password'].encode()
                )
        except FileNotFoundError:
            # Generate new key pair
            self.private_key = rsa.generate_private_key(
                public_exponent=65537,
                key_size=4096
            )
            self._save_keys()
        
        self.public_key = self.private_key.public_key()
    
    def authenticate_user(self, username: str, password: str) -> Dict:
        """Authenticate user and return tokens"""
        # Get user from database
        user = self._get_user_by_username(username)
        if not user:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid credentials"
            )
        
        # Verify password
        try:
            self.password_hasher.verify(user['password_hash'], password)
        except:
            # Log failed attempt
            self._record_failed_attempt(username)
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid credentials"
            )
        
        # Check if account is locked
        if self._is_account_locked(username):
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Account temporarily locked due to multiple failed attempts"
            )
        
        # Generate tokens
        access_token = self._create_access_token(user)
        refresh_token = self._create_refresh_token(user)
        
        # Clear failed attempts
        self._clear_failed_attempts(username)
        
        return {
            'access_token': access_token,
            'refresh_token': refresh_token,
            'token_type': 'Bearer',
            'expires_in': self.config['access_token_expire_minutes'] * 60
        }
    
    def _create_access_token(self, user: Dict) -> str:
        """Create JWT access token"""
        now = datetime.utcnow()
        jti = secrets.token_urlsafe(32)
        
        payload = {
            'sub': str(user['id']),
            'exp': now + timedelta(minutes=self.config['access_token_expire_minutes']),
            'iat': now,
            'jti': jti,
            'type': 'access',
            'username': user['username'],
            'email': user['email'],
            'roles': user['roles'],
            'permissions': self._get_permissions_for_roles(user['roles']),
            'organisation_id': str(user['organisation_id'])
        }
        
        token = jwt.encode(
            payload,
            self.private_key,
            algorithm='RS256'
        )
        
        # Store JTI for revocation capability
        self.redis_client.setex(
            f"jwt:access:{jti}",
            self.config['access_token_expire_minutes'] * 60,
            'valid'
        )
        
        return token
    
    def _create_refresh_token(self, user: Dict) -> str:
        """Create JWT refresh token"""
        now = datetime.utcnow()
        jti = secrets.token_urlsafe(32)
        
        payload = {
            'sub': str(user['id']),
            'exp': now + timedelta(days=self.config['refresh_token_expire_days']),
            'iat': now,
            'jti': jti,
            'type': 'refresh'
        }
        
        token = jwt.encode(
            payload,
            self.private_key,
            algorithm='RS256'
        )
        
        # Store refresh token
        self.redis_client.setex(
            f"jwt:refresh:{jti}",
            self.config['refresh_token_expire_days'] * 86400,
            str(user['id'])
        )
        
        return token
    
    def verify_token(self, credentials: HTTPAuthorizationCredentials) -> TokenPayload:
        """Verify and decode JWT token"""
        token = credentials.credentials
        
        try:
            # Decode token
            payload = jwt.decode(
                token,
                self.public_key,
                algorithms=['RS256']
            )
            
            # Check if token is revoked
            jti = payload.get('jti')
            if not self.redis_client.exists(f"jwt:access:{jti}"):
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Token has been revoked"
                )
            
            return TokenPayload(**payload)
            
        except jwt.ExpiredSignatureError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Token has expired"
            )
        except jwt.InvalidTokenError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token"
            )
    
    def refresh_access_token(self, refresh_token: str) -> Dict:
        """Use refresh token to get new access token"""
        try:
            # Decode refresh token
            payload = jwt.decode(
                refresh_token,
                self.public_key,
                algorithms=['RS256']
            )
            
            # Verify it's a refresh token
            if payload.get('type') != 'refresh':
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail="Invalid token type"
                )
            
            # Check if refresh token is valid
            jti = payload.get('jti')
            user_id = self.redis_client.get(f"jwt:refresh:{jti}")
            if not user_id:
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Invalid refresh token"
                )
            
            # Get updated user data
            user = self._get_user_by_id(user_id)
            if not user:
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="User not found"
                )
            
            # Generate new access token
            access_token = self._create_access_token(user)
            
            return {
                'access_token': access_token,
                'token_type': 'Bearer',
                'expires_in': self.config['access_token_expire_minutes'] * 60
            }
            
        except jwt.InvalidTokenError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid refresh token"
            )
    
    def revoke_token(self, token: str):
        """Revoke a token"""
        try:
            payload = jwt.decode(
                token,
                self.public_key,
                algorithms=['RS256']
            )
            
            jti = payload.get('jti')
            token_type = payload.get('type', 'access')
            
            # Remove from Redis
            self.redis_client.delete(f"jwt:{token_type}:{jti}")
            
        except jwt.InvalidTokenError:
            pass  # Token already invalid
```

### Security Middleware
```python
from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware
import time

class AuthenticationMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, auth_service: AuthenticationService):
        super().__init__(app)
        self.auth_service = auth_service
        self.public_paths = {'/docs', '/openapi.json', '/health', '/api/v1/auth/login'}
    
    async def dispatch(self, request: Request, call_next):
        # Skip authentication for public paths
        if request.url.path in self.public_paths:
            return await call_next(request)
        
        # Extract token from header
        auth_header = request.headers.get('Authorization')
        if not auth_header or not auth_header.startswith('Bearer '):
            return JSONResponse(
                status_code=401,
                content={'detail': 'Missing authentication token'}
            )
        
        try:
            # Verify token
            token = auth_header.split(' ')[1]
            credentials = HTTPAuthorizationCredentials(
                scheme='Bearer',
                credentials=token
            )
            token_payload = self.auth_service.verify_token(credentials)
            
            # Add user context to request
            request.state.user = token_payload
            
            # Process request
            response = await call_next(request)
            
            return response
            
        except HTTPException as e:
            return JSONResponse(
                status_code=e.status_code,
                content={'detail': e.detail}
            )
```

### Rate Limiting
```python
from typing import Dict
import time

class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.limits = {
            'login': {'requests': 5, 'window': 300},  # 5 attempts per 5 minutes
            'api': {'requests': 100, 'window': 60}    # 100 requests per minute
        }
    
    def check_rate_limit(self, key: str, limit_type: str = 'api') -> bool:
        """Check if request should be rate limited"""
        limit = self.limits.get(limit_type)
        if not limit:
            return False
        
        # Use sliding window counter
        now = time.time()
        window_start = now - limit['window']
        
        # Remove old entries
        self.redis.zremrangebyscore(f"rate:{limit_type}:{key}", 0, window_start)
        
        # Count requests in window
        count = self.redis.zcard(f"rate:{limit_type}:{key}")
        
        if count >= limit['requests']:
            return True
        
        # Add current request
        self.redis.zadd(f"rate:{limit_type}:{key}", {str(now): now})
        self.redis.expire(f"rate:{limit_type}:{key}", limit['window'])
        
        return False
    
    def get_retry_after(self, key: str, limit_type: str = 'api') -> int:
        """Get seconds until rate limit resets"""
        limit = self.limits.get(limit_type)
        if not limit:
            return 0
        
        # Get oldest entry
        oldest = self.redis.zrange(f"rate:{limit_type}:{key}", 0, 0, withscores=True)
        if oldest:
            oldest_time = oldest[0][1]
            retry_after = int(oldest_time + limit['window'] - time.time())
            return max(0, retry_after)
        
        return 0
```

## Testing Requirements
1. **Security Tests**
   - Token signature verification
   - Expired token handling
   - Invalid token rejection
   - Rate limiting enforcement

2. **Integration Tests**
   - Login flow end-to-end
   - Token refresh workflow
   - Concurrent authentication
   - Redis failover handling

3. **Performance Tests**
   - Authentication response time
   - Token generation throughput
   - Concurrent user limits
   - Redis connection pooling

## Definition of Done
- [ ] JWT authentication implemented with RS256
- [ ] Access and refresh tokens working
- [ ] Token revocation via Redis
- [ ] Rate limiting on login endpoint
- [ ] Failed attempt tracking
- [ ] Account lockout after failures
- [ ] Authentication middleware protecting all endpoints
- [ ] Performance targets met (< 200ms)