# User Story 02.3: LDAP Integration

## Story
As an **enterprise user**, I want to **login using my corporate LDAP credentials** so that **I don't need to manage separate passwords**.

## Acceptance Criteria
```gherkin
Feature: LDAP Authentication Integration
  As an enterprise user
  I want to authenticate using LDAP credentials
  So that I can use my corporate identity

  Scenario: LDAP authentication
    Given I have valid LDAP credentials
    When I login with my corporate username and password
    Then I should be authenticated against the LDAP server
    And my user profile should be created or updated locally
    And I should receive JWT tokens as normal
    And my LDAP groups should map to system roles

  Scenario: LDAP server failover
    Given multiple LDAP servers are configured
    When the primary LDAP server is unavailable
    Then authentication should failover to secondary servers
    And there should be no noticeable delay for users
    And failover events should be logged

  Scenario: Group-to-role mapping
    Given LDAP groups are mapped to system roles
    When I authenticate via LDAP
    Then my LDAP groups should be retrieved
    And they should be mapped to corresponding system roles
    And unmapped groups should be ignored
    And changes in LDAP groups should reflect on next login

  Scenario: Mixed authentication
    Given both local and LDAP authentication are enabled
    When users attempt to login
    Then LDAP users should authenticate via LDAP
    And local users should authenticate locally
    And service accounts should always use local authentication
```

## Technical Details

### LDAP Configuration
```python
from ldap3 import Server, Connection, ALL, NTLM, SIMPLE
from ldap3.core.exceptions import LDAPException
from typing import Dict, List, Optional, Tuple
import ssl
from dataclasses import dataclass
import re

@dataclass
class LDAPConfig:
    servers: List[str]
    base_dn: str
    bind_dn: Optional[str]
    bind_password: Optional[str]
    user_search_base: str
    user_search_filter: str
    group_search_base: str
    group_search_filter: str
    use_ssl: bool = True
    use_tls: bool = False
    verify_cert: bool = True
    timeout: int = 10
    pool_size: int = 10
    group_attribute: str = 'memberOf'
    username_attribute: str = 'sAMAccountName'
    email_attribute: str = 'mail'
    display_name_attribute: str = 'displayName'

class LDAPService:
    def __init__(self, config: LDAPConfig, role_mapping: Dict[str, str]):
        self.config = config
        self.role_mapping = role_mapping  # LDAP group -> system role
        self.servers = self._initialize_servers()
        self._connection_pool = []
        
    def _initialize_servers(self) -> List[Server]:
        """Initialize LDAP server connections"""
        servers = []
        
        for server_url in self.config.servers:
            tls = ssl.create_default_context()
            if not self.config.verify_cert:
                tls.check_hostname = False
                tls.verify_mode = ssl.CERT_NONE
            
            server = Server(
                server_url,
                use_ssl=self.config.use_ssl,
                tls=tls,
                get_info=ALL,
                connect_timeout=self.config.timeout
            )
            servers.append(server)
        
        return servers
    
    def authenticate_user(self, username: str, password: str) -> Optional[Dict]:
        """Authenticate user against LDAP"""
        # Try each server until successful
        for server in self.servers:
            try:
                user_info = self._authenticate_against_server(
                    server, username, password
                )
                if user_info:
                    return user_info
            except LDAPException as e:
                self.logger.warning(
                    f"LDAP authentication failed on {server.host}: {str(e)}"
                )
                continue
        
        return None
    
    def _authenticate_against_server(self, server: Server, 
                                   username: str, password: str) -> Optional[Dict]:
        """Authenticate against specific LDAP server"""
        # Escape special characters in username
        safe_username = self._escape_ldap_filter(username)
        
        # Build user search filter
        user_filter = self.config.user_search_filter.format(
            username=safe_username
        )
        
        try:
            # First bind with service account to search for user
            if self.config.bind_dn:
                conn = Connection(
                    server,
                    user=self.config.bind_dn,
                    password=self.config.bind_password,
                    authentication=SIMPLE,
                    auto_bind=True,
                    raise_exceptions=True
                )
            else:
                # Anonymous bind
                conn = Connection(
                    server,
                    auto_bind=True,
                    raise_exceptions=True
                )
            
            # Search for user
            conn.search(
                search_base=self.config.user_search_base,
                search_filter=user_filter,
                attributes=[
                    self.config.username_attribute,
                    self.config.email_attribute,
                    self.config.display_name_attribute,
                    self.config.group_attribute,
                    'distinguishedName'
                ]
            )
            
            if not conn.entries:
                return None
            
            user_entry = conn.entries[0]
            user_dn = user_entry.distinguishedName.value
            
            # Close search connection
            conn.unbind()
            
            # Attempt to bind as the user
            user_conn = Connection(
                server,
                user=user_dn,
                password=password,
                authentication=SIMPLE,
                auto_bind=True,
                raise_exceptions=True
            )
            
            # Authentication successful - get user info
            user_info = {
                'username': getattr(user_entry, self.config.username_attribute).value,
                'email': getattr(user_entry, self.config.email_attribute).value,
                'display_name': getattr(user_entry, self.config.display_name_attribute).value,
                'dn': user_dn,
                'groups': self._get_user_groups(user_entry),
                'ldap_server': server.host
            }
            
            # Map LDAP groups to system roles
            user_info['roles'] = self._map_groups_to_roles(user_info['groups'])
            
            # Close user connection
            user_conn.unbind()
            
            return user_info
            
        except LDAPException:
            return None
    
    def _get_user_groups(self, user_entry) -> List[str]:
        """Extract user groups from LDAP entry"""
        groups = []
        
        # Get groups from memberOf attribute
        member_of = getattr(user_entry, self.config.group_attribute, None)
        if member_of:
            if isinstance(member_of.value, list):
                groups.extend(member_of.value)
            else:
                groups.append(member_of.value)
        
        # Extract CN from DN for each group
        group_names = []
        for group_dn in groups:
            match = re.search(r'CN=([^,]+)', group_dn, re.IGNORECASE)
            if match:
                group_names.append(match.group(1))
        
        return group_names
    
    def _map_groups_to_roles(self, ldap_groups: List[str]) -> List[str]:
        """Map LDAP groups to system roles"""
        roles = []
        
        for group in ldap_groups:
            # Check exact match
            if group in self.role_mapping:
                roles.append(self.role_mapping[group])
            # Check pattern match
            else:
                for pattern, role in self.role_mapping.items():
                    if '*' in pattern:
                        regex = pattern.replace('*', '.*')
                        if re.match(regex, group, re.IGNORECASE):
                            roles.append(role)
                            break
        
        # Default role if no mappings
        if not roles:
            roles.append('viewer')  # Default minimal access
        
        return list(set(roles))  # Remove duplicates
    
    def _escape_ldap_filter(self, value: str) -> str:
        """Escape special characters in LDAP filter"""
        # LDAP filter escape rules
        escape_chars = {
            '\\': r'\5c',
            '*': r'\2a',
            '(': r'\28',
            ')': r'\29',
            '\x00': r'\00',
            '/': r'\2f'
        }
        
        for char, escaped in escape_chars.items():
            value = value.replace(char, escaped)
        
        return value
    
    def sync_user_groups(self, username: str) -> Optional[List[str]]:
        """Synchronise user groups from LDAP"""
        # Use service account to query current groups
        for server in self.servers:
            try:
                conn = Connection(
                    server,
                    user=self.config.bind_dn,
                    password=self.config.bind_password,
                    auto_bind=True
                )
                
                # Search for user
                user_filter = self.config.user_search_filter.format(
                    username=self._escape_ldap_filter(username)
                )
                
                conn.search(
                    search_base=self.config.user_search_base,
                    search_filter=user_filter,
                    attributes=[self.config.group_attribute]
                )
                
                if conn.entries:
                    groups = self._get_user_groups(conn.entries[0])
                    roles = self._map_groups_to_roles(groups)
                    conn.unbind()
                    return roles
                
                conn.unbind()
                
            except LDAPException:
                continue
        
        return None
```

### Integration with Authentication Service
```python
class HybridAuthenticationService(AuthenticationService):
    """Authentication service supporting both local and LDAP authentication"""
    
    def __init__(self, config: Dict, ldap_service: Optional[LDAPService] = None):
        super().__init__(config)
        self.ldap_service = ldap_service
        self.user_service = UserService()
    
    def authenticate_user(self, username: str, password: str) -> Dict:
        """Authenticate user via LDAP or local database"""
        # Check if user should use LDAP
        user = self.user_service.get_user_by_username(username)
        
        if user and user.get('auth_type') == 'ldap' and self.ldap_service:
            # LDAP authentication
            ldap_info = self.ldap_service.authenticate_user(username, password)
            
            if ldap_info:
                # Update or create local user record
                user = self._sync_ldap_user(username, ldap_info)
                
                # Generate tokens
                return self._generate_tokens(user)
            else:
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Invalid credentials"
                )
        else:
            # Local authentication
            return super().authenticate_user(username, password)
    
    def _sync_ldap_user(self, username: str, ldap_info: Dict) -> Dict:
        """Synchronise LDAP user with local database"""
        existing_user = self.user_service.get_user_by_username(username)
        
        if existing_user:
            # Update existing user
            updates = {
                'email': ldap_info['email'],
                'display_name': ldap_info['display_name'],
                'ldap_dn': ldap_info['dn'],
                'last_ldap_sync': datetime.utcnow()
            }
            
            # Update roles if changed
            current_roles = set(existing_user['roles'])
            ldap_roles = set(ldap_info['roles'])
            
            if current_roles != ldap_roles:
                updates['roles'] = list(ldap_roles)
                self._log_role_change(username, current_roles, ldap_roles)
            
            user = self.user_service.update_user(existing_user['id'], updates)
        else:
            # Create new user
            user = self.user_service.create_user({
                'username': username,
                'email': ldap_info['email'],
                'display_name': ldap_info['display_name'],
                'auth_type': 'ldap',
                'ldap_dn': ldap_info['dn'],
                'roles': ldap_info['roles'],
                'organisation_id': self._get_default_organisation(),
                'created_at': datetime.utcnow(),
                'last_ldap_sync': datetime.utcnow()
            })
        
        return user
```

### LDAP Configuration API
```python
@router.post("/api/v1/admin/ldap/test")
@require_permission(Permission.SYSTEM_CONFIG)
async def test_ldap_connection(
    config: LDAPConfig,
    current_user: TokenPayload = Depends(get_current_user)
):
    """Test LDAP configuration"""
    try:
        ldap_service = LDAPService(config, {})
        
        # Test connection to each server
        results = []
        for server in ldap_service.servers:
            try:
                conn = Connection(
                    server,
                    user=config.bind_dn,
                    password=config.bind_password,
                    auto_bind=True
                )
                conn.unbind()
                results.append({
                    'server': server.host,
                    'status': 'success',
                    'message': 'Connection successful'
                })
            except Exception as e:
                results.append({
                    'server': server.host,
                    'status': 'failed',
                    'message': str(e)
                })
        
        return {
            'overall_status': 'success' if any(r['status'] == 'success' for r in results) else 'failed',
            'servers': results
        }
        
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"LDAP configuration test failed: {str(e)}"
        )

@router.post("/api/v1/admin/ldap/mappings")
@require_permission(Permission.SYSTEM_CONFIG)
async def configure_ldap_mappings(
    mappings: Dict[str, str],
    current_user: TokenPayload = Depends(get_current_user)
):
    """Configure LDAP group to role mappings"""
    # Validate all target roles exist
    role_service = get_role_service()
    valid_roles = [r.name for r in role_service.list_roles()]
    
    for ldap_group, role in mappings.items():
        if role not in valid_roles:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"Invalid role: {role}"
            )
    
    # Save mappings
    config_service = get_config_service()
    config_service.set('ldap.group_mappings', mappings)
    
    return {
        'message': 'LDAP mappings configured successfully',
        'mappings': mappings
    }
```

### LDAP Health Monitoring
```python
class LDAPHealthMonitor:
    def __init__(self, ldap_service: LDAPService):
        self.ldap_service = ldap_service
        self.metrics = MetricsCollector()
    
    async def check_ldap_health(self):
        """Monitor LDAP server health"""
        health_status = {
            'healthy_servers': 0,
            'total_servers': len(self.ldap_service.servers),
            'server_status': []
        }
        
        for server in self.ldap_service.servers:
            start_time = time.time()
            try:
                conn = Connection(
                    server,
                    user=self.ldap_service.config.bind_dn,
                    password=self.ldap_service.config.bind_password,
                    auto_bind=True,
                    receive_timeout=5
                )
                
                # Test search
                conn.search(
                    search_base=self.ldap_service.config.base_dn,
                    search_filter='(objectClass=*)',
                    attributes=['1.1'],
                    size_limit=1
                )
                
                conn.unbind()
                
                response_time = (time.time() - start_time) * 1000
                
                health_status['server_status'].append({
                    'server': server.host,
                    'status': 'healthy',
                    'response_time_ms': response_time
                })
                health_status['healthy_servers'] += 1
                
                # Record metrics
                self.metrics.histogram(
                    'ldap.server.response_time',
                    response_time,
                    tags={'server': server.host}
                )
                
            except Exception as e:
                health_status['server_status'].append({
                    'server': server.host,
                    'status': 'unhealthy',
                    'error': str(e)
                })
                
                # Record failure
                self.metrics.increment(
                    'ldap.server.failures',
                    tags={'server': server.host}
                )
        
        # Overall health
        health_status['overall_health'] = (
            'healthy' if health_status['healthy_servers'] > 0 else 'unhealthy'
        )
        
        return health_status
```

## Testing Requirements
1. **Integration Tests**
   - LDAP server connectivity
   - Authentication flow
   - Group mapping accuracy
   - Failover behaviour

2. **Security Tests**
   - LDAP injection prevention
   - SSL/TLS verification
   - Bind credential security
   - Password handling

3. **Performance Tests**
   - Authentication response time
   - Concurrent LDAP queries
   - Connection pooling
   - Failover timing

## Definition of Done
- [ ] LDAP authentication working with AD and OpenLDAP
- [ ] Multiple server failover implemented
- [ ] Group to role mapping configured
- [ ] SSL/TLS connections enforced
- [ ] User synchronisation on login
- [ ] Service account binding
- [ ] Health monitoring implemented
- [ ] Configuration UI/API available