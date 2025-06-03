# User Story 02.2: Role-Based Access Control

## Story
As a **system administrator**, I want to **manage user permissions through roles** so that **users only have access to the functions they need**.

## Acceptance Criteria
```gherkin
Feature: Role-Based Access Control
  As a system administrator
  I want to assign roles and permissions to users
  So that access is properly controlled

  Scenario: Predefined role assignment
    Given I have predefined roles (Admin, Operator, Developer, Auditor, Viewer)
    When I assign a role to a user
    Then the user should have all permissions associated with that role
    And the permissions should be enforced on API calls
    And role changes should take effect immediately

  Scenario: Custom role creation
    Given I am an administrator
    When I create a custom role with specific permissions
    Then the role should be available for assignment
    And I should be able to modify the role's permissions
    And users with the role should have the exact permissions specified

  Scenario: Permission enforcement
    Given a user with specific role permissions
    When they attempt to access a protected resource
    Then access should be granted only if they have the required permission
    And denied requests should return 403 Forbidden
    And the denial should be logged for audit

  Scenario: Role hierarchy
    Given roles can inherit from other roles
    When I create a role hierarchy
    Then child roles should have all parent permissions
    And permission changes should cascade appropriately
    And circular inheritance should be prevented
```

## Technical Details

### RBAC Model Implementation
```python
from typing import List, Dict, Set, Optional
from enum import Enum
from pydantic import BaseModel
from fastapi import HTTPException, status
import uuid
from datetime import datetime

class Permission(str, Enum):
    """System permissions"""
    # Certificate operations
    CERT_CREATE = "certificates:create"
    CERT_READ = "certificates:read"
    CERT_UPDATE = "certificates:update"
    CERT_DELETE = "certificates:delete"
    CERT_REVOKE = "certificates:revoke"
    CERT_EXPORT = "certificates:export"
    
    # CA operations
    CA_CREATE = "ca:create"
    CA_READ = "ca:read"
    CA_UPDATE = "ca:update"
    CA_DELETE = "ca:delete"
    CA_SIGN = "ca:sign"
    
    # User management
    USER_CREATE = "users:create"
    USER_READ = "users:read"
    USER_UPDATE = "users:update"
    USER_DELETE = "users:delete"
    USER_ASSIGN_ROLE = "users:assign_role"
    
    # Role management
    ROLE_CREATE = "roles:create"
    ROLE_READ = "roles:read"
    ROLE_UPDATE = "roles:update"
    ROLE_DELETE = "roles:delete"
    
    # Audit operations
    AUDIT_READ = "audit:read"
    AUDIT_EXPORT = "audit:export"
    
    # System operations
    SYSTEM_CONFIG = "system:config"
    SYSTEM_BACKUP = "system:backup"
    SYSTEM_RESTORE = "system:restore"

class PredefinedRole(str, Enum):
    """Built-in system roles"""
    ADMIN = "admin"
    OPERATOR = "operator"
    DEVELOPER = "developer"
    AUDITOR = "auditor"
    VIEWER = "viewer"

class Role(BaseModel):
    id: uuid.UUID
    name: str
    description: str
    permissions: List[Permission]
    parent_roles: List[uuid.UUID] = []
    is_system: bool = False
    created_at: datetime
    updated_at: datetime

class RBACService:
    def __init__(self, db_session):
        self.db = db_session
        self._initialize_predefined_roles()
    
    def _initialize_predefined_roles(self):
        """Create predefined system roles"""
        predefined_roles = {
            PredefinedRole.ADMIN: {
                'description': 'Full system administration',
                'permissions': [p for p in Permission]  # All permissions
            },
            PredefinedRole.OPERATOR: {
                'description': 'Certificate operations and management',
                'permissions': [
                    Permission.CERT_CREATE,
                    Permission.CERT_READ,
                    Permission.CERT_UPDATE,
                    Permission.CERT_REVOKE,
                    Permission.CERT_EXPORT,
                    Permission.CA_READ,
                    Permission.CA_SIGN,
                    Permission.USER_READ,
                    Permission.AUDIT_READ
                ]
            },
            PredefinedRole.DEVELOPER: {
                'description': 'Certificate creation and reading',
                'permissions': [
                    Permission.CERT_CREATE,
                    Permission.CERT_READ,
                    Permission.CERT_EXPORT,
                    Permission.CA_READ
                ]
            },
            PredefinedRole.AUDITOR: {
                'description': 'Audit and compliance access',
                'permissions': [
                    Permission.CERT_READ,
                    Permission.CA_READ,
                    Permission.USER_READ,
                    Permission.ROLE_READ,
                    Permission.AUDIT_READ,
                    Permission.AUDIT_EXPORT
                ]
            },
            PredefinedRole.VIEWER: {
                'description': 'Read-only access',
                'permissions': [
                    Permission.CERT_READ,
                    Permission.CA_READ,
                    Permission.USER_READ,
                    Permission.ROLE_READ
                ]
            }
        }
        
        for role_name, config in predefined_roles.items():
            self._create_system_role(role_name, config)
    
    def create_custom_role(self, name: str, description: str, 
                          permissions: List[Permission], 
                          parent_roles: List[uuid.UUID] = None) -> Role:
        """Create a custom role"""
        # Validate no circular inheritance
        if parent_roles:
            self._validate_role_hierarchy(parent_roles)
        
        # Create role
        role = Role(
            id=uuid.uuid4(),
            name=name,
            description=description,
            permissions=permissions,
            parent_roles=parent_roles or [],
            is_system=False,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow()
        )
        
        # Save to database
        self.db.save_role(role)
        
        # Clear permission cache
        self._clear_permission_cache()
        
        return role
    
    def get_effective_permissions(self, role_ids: List[uuid.UUID]) -> Set[Permission]:
        """Get all permissions for a set of roles including inheritance"""
        permissions = set()
        
        # Check cache first
        cache_key = f"perms:{':'.join(sorted(str(id) for id in role_ids))}"
        cached = self._get_cached_permissions(cache_key)
        if cached:
            return cached
        
        # Process each role
        processed_roles = set()
        roles_to_process = list(role_ids)
        
        while roles_to_process:
            role_id = roles_to_process.pop()
            
            if role_id in processed_roles:
                continue
            
            processed_roles.add(role_id)
            
            # Get role
            role = self.db.get_role(role_id)
            if not role:
                continue
            
            # Add direct permissions
            permissions.update(role.permissions)
            
            # Add parent roles to process
            roles_to_process.extend(role.parent_roles)
        
        # Cache result
        self._cache_permissions(cache_key, permissions)
        
        return permissions
    
    def check_permission(self, user_roles: List[uuid.UUID], 
                        required_permission: Permission) -> bool:
        """Check if user roles have required permission"""
        effective_permissions = self.get_effective_permissions(user_roles)
        return required_permission in effective_permissions
    
    def assign_role_to_user(self, user_id: uuid.UUID, role_id: uuid.UUID):
        """Assign a role to a user"""
        # Verify role exists
        role = self.db.get_role(role_id)
        if not role:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="Role not found"
            )
        
        # Add role to user
        self.db.add_user_role(user_id, role_id)
        
        # Clear user's permission cache
        self._clear_user_permission_cache(user_id)
    
    def remove_role_from_user(self, user_id: uuid.UUID, role_id: uuid.UUID):
        """Remove a role from a user"""
        self.db.remove_user_role(user_id, role_id)
        self._clear_user_permission_cache(user_id)
    
    def update_role_permissions(self, role_id: uuid.UUID, 
                               permissions: List[Permission]):
        """Update permissions for a role"""
        role = self.db.get_role(role_id)
        if not role:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="Role not found"
            )
        
        if role.is_system:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Cannot modify system roles"
            )
        
        # Update permissions
        role.permissions = permissions
        role.updated_at = datetime.utcnow()
        
        self.db.update_role(role)
        
        # Clear all permission caches
        self._clear_permission_cache()
```

### Permission Decorators
```python
from functools import wraps
from typing import Union, List
from fastapi import Depends, HTTPException, status

def require_permission(permission: Union[Permission, List[Permission]]):
    """Decorator to require specific permissions for endpoint access"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Get current user from request context
            request = kwargs.get('request')
            if not request or not hasattr(request.state, 'user'):
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Authentication required"
                )
            
            user = request.state.user
            
            # Get RBAC service
            rbac_service = kwargs.get('rbac_service')
            if not rbac_service:
                rbac_service = get_rbac_service()
            
            # Check permissions
            required_perms = [permission] if isinstance(permission, Permission) else permission
            
            for perm in required_perms:
                if not rbac_service.check_permission(user.roles, perm):
                    # Log denied access
                    audit_logger.warning(
                        f"Permission denied: User {user.sub} lacks {perm}",
                        extra={
                            'user_id': user.sub,
                            'required_permission': perm,
                            'user_roles': user.roles,
                            'endpoint': request.url.path
                        }
                    )
                    
                    raise HTTPException(
                        status_code=status.HTTP_403_FORBIDDEN,
                        detail=f"Permission denied: {perm} required"
                    )
            
            # Permission granted
            return await func(*args, **kwargs)
        
        return wrapper
    return decorator

def require_any_permission(permissions: List[Permission]):
    """Require at least one of the specified permissions"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            request = kwargs.get('request')
            if not request or not hasattr(request.state, 'user'):
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Authentication required"
                )
            
            user = request.state.user
            rbac_service = kwargs.get('rbac_service', get_rbac_service())
            
            # Check if user has any of the required permissions
            for perm in permissions:
                if rbac_service.check_permission(user.roles, perm):
                    return await func(*args, **kwargs)
            
            # No permission matched
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"One of these permissions required: {', '.join(permissions)}"
            )
        
        return wrapper
    return decorator
```

### API Endpoints
```python
from fastapi import APIRouter, Depends
from typing import List

router = APIRouter(prefix="/api/v1/roles", tags=["roles"])

@router.get("/", response_model=List[Role])
@require_permission(Permission.ROLE_READ)
async def list_roles(
    rbac_service: RBACService = Depends(get_rbac_service),
    current_user: TokenPayload = Depends(get_current_user)
):
    """List all roles"""
    return rbac_service.list_roles()

@router.post("/", response_model=Role)
@require_permission(Permission.ROLE_CREATE)
async def create_role(
    role_request: RoleCreateRequest,
    rbac_service: RBACService = Depends(get_rbac_service),
    current_user: TokenPayload = Depends(get_current_user)
):
    """Create a custom role"""
    return rbac_service.create_custom_role(
        name=role_request.name,
        description=role_request.description,
        permissions=role_request.permissions,
        parent_roles=role_request.parent_roles
    )

@router.put("/{role_id}/permissions")
@require_permission(Permission.ROLE_UPDATE)
async def update_role_permissions(
    role_id: uuid.UUID,
    permissions: List[Permission],
    rbac_service: RBACService = Depends(get_rbac_service),
    current_user: TokenPayload = Depends(get_current_user)
):
    """Update role permissions"""
    rbac_service.update_role_permissions(role_id, permissions)
    return {"message": "Permissions updated successfully"}

@router.post("/users/{user_id}/roles/{role_id}")
@require_permission(Permission.USER_ASSIGN_ROLE)
async def assign_role(
    user_id: uuid.UUID,
    role_id: uuid.UUID,
    rbac_service: RBACService = Depends(get_rbac_service),
    current_user: TokenPayload = Depends(get_current_user)
):
    """Assign a role to a user"""
    rbac_service.assign_role_to_user(user_id, role_id)
    return {"message": "Role assigned successfully"}

@router.get("/check-permission")
async def check_user_permission(
    permission: Permission,
    rbac_service: RBACService = Depends(get_rbac_service),
    current_user: TokenPayload = Depends(get_current_user)
):
    """Check if current user has a specific permission"""
    has_permission = rbac_service.check_permission(
        current_user.roles,
        permission
    )
    return {
        "permission": permission,
        "granted": has_permission
    }
```

### Permission Caching
```python
import redis
import json
from typing import Set

class PermissionCache:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.ttl = 300  # 5 minutes
    
    def get_permissions(self, cache_key: str) -> Optional[Set[Permission]]:
        """Get cached permissions"""
        data = self.redis.get(f"perm_cache:{cache_key}")
        if data:
            return set(Permission(p) for p in json.loads(data))
        return None
    
    def set_permissions(self, cache_key: str, permissions: Set[Permission]):
        """Cache permissions"""
        data = json.dumps([p.value for p in permissions])
        self.redis.setex(
            f"perm_cache:{cache_key}",
            self.ttl,
            data
        )
    
    def clear_user_cache(self, user_id: str):
        """Clear all permission caches for a user"""
        pattern = f"perm_cache:*{user_id}*"
        for key in self.redis.scan_iter(match=pattern):
            self.redis.delete(key)
    
    def clear_all_cache(self):
        """Clear all permission caches"""
        pattern = "perm_cache:*"
        for key in self.redis.scan_iter(match=pattern):
            self.redis.delete(key)
```

## Testing Requirements
1. **Permission Tests**
   - Role assignment verification
   - Permission inheritance testing
   - Circular inheritance prevention
   - Permission denial logging

2. **Integration Tests**
   - End-to-end permission checks
   - Role modification effects
   - Cache invalidation
   - Concurrent role updates

3. **Performance Tests**
   - Permission check speed
   - Cache effectiveness
   - Role hierarchy depth impact
   - Bulk permission checks

## Definition of Done
- [ ] Predefined roles created and configured
- [ ] Custom role creation working
- [ ] Permission inheritance implemented
- [ ] Permission decorators protecting endpoints
- [ ] Role assignment to users functional
- [ ] Permission caching for performance
- [ ] Audit logging for permission denials
- [ ] Circular inheritance prevention