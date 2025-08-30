# Authentication vs Authorization

## What are Authentication and Authorization?

Authentication and Authorization are two fundamental security concepts that work together to protect systems and data, but they serve distinctly different purposes. Think of authentication like showing your ID at a nightclub entrance - the bouncer needs to verify that you are who you claim to be by checking your identification. Authorization is like the VIP list that determines what areas of the club you can access once you're inside - even though you've proven your identity, you might only be allowed in certain sections based on your membership level or special privileges.

Authentication answers the question "Who are you?" while authorization answers "What are you allowed to do?" These concepts are often confused because they work so closely together, but understanding their differences is crucial for designing secure systems. Authentication must happen first to establish identity, and then authorization determines what that authenticated identity is permitted to access or perform.

## Authentication: Proving Identity

### What Authentication Accomplishes

Authentication is the process of verifying that users or systems are who they claim to be. This verification can be based on something you know (like a password), something you have (like a phone or security token), something you are (like biometric data), or a combination of these factors. The goal is to establish a trusted identity that the system can then use to make decisions about access and permissions.

Successful authentication creates a security context or session that represents the verified identity. This context typically includes information about the user, when they authenticated, and potentially additional metadata about their authentication method or device. This context is then used throughout the user's session to make authorization decisions without requiring repeated authentication.

### Types of Authentication

**Password-based authentication** is the most common form, where users provide a username and password combination. While simple to implement and understand, password-based authentication has well-known vulnerabilities including password reuse, weak passwords, and susceptibility to various attacks like brute force, dictionary attacks, and credential stuffing.

**Multi-factor authentication (MFA)** combines multiple authentication factors to provide stronger security. Common implementations include SMS codes, authenticator apps, hardware tokens, or biometric verification in addition to passwords. MFA significantly improves security by requiring attackers to compromise multiple independent factors.

**Single Sign-On (SSO)** allows users to authenticate once and access multiple related systems without re-authenticating. SSO improves user experience while potentially strengthening security by centralizing authentication and allowing for stronger authentication methods and better monitoring.

**Certificate-based authentication** uses digital certificates to verify identity, commonly used for system-to-system authentication or in high-security environments. Certificates provide strong cryptographic proof of identity and can be managed centrally through Certificate Authorities.

### Authentication Challenges

Modern authentication faces numerous challenges including credential management, user experience, and security threats. Users struggle with password fatigue, leading to weak passwords or password reuse across systems. Organizations must balance security requirements with usability, as overly complex authentication processes can lead to user frustration and workarounds that actually reduce security.

Distributed systems add complexity to authentication, requiring secure token management, session handling across services, and consistent authentication policies. Mobile and IoT devices introduce additional challenges around device trust, certificate management, and handling intermittent connectivity.

## Authorization: Controlling Access

### What Authorization Accomplishes

Authorization determines what authenticated users are allowed to do within a system. This includes access to specific resources, permission to perform certain operations, and the scope of data they can view or modify. Authorization policies are typically based on the user's role, attributes, or relationship to the requested resource.

Effective authorization systems provide fine-grained control over system access while remaining manageable and understandable. They must handle complex scenarios like hierarchical permissions, temporary access grants, and context-dependent access rules while maintaining good performance and auditability.

### Authorization Models

**Role-Based Access Control (RBAC)** assigns permissions to roles rather than individual users, and users are assigned to one or more roles. This model simplifies permission management by grouping related permissions together and allowing administrators to manage access by assigning appropriate roles to users. RBAC works well for organizations with clear hierarchical structures and well-defined job functions.

**Attribute-Based Access Control (ABAC)** makes authorization decisions based on attributes of the user, resource, action, and environment. This model provides more flexibility than RBAC by allowing complex policies that consider multiple factors like time of day, user location, resource sensitivity, and current security context. ABAC is more complex to implement but can handle sophisticated authorization requirements.

**Access Control Lists (ACLs)** specify which users or groups have what permissions on specific resources. ACLs provide direct, resource-centric access control but can become difficult to manage at scale. They're often used for file systems and simple resource protection scenarios.

**Policy-Based Access Control** uses externalized policy engines to make authorization decisions based on complex rules and policies. This approach separates authorization logic from application code, making it easier to manage and audit access policies across multiple systems.

### Authorization Enforcement

Authorization can be enforced at multiple layers within a system architecture. **Application-level authorization** is implemented within the application code and provides fine-grained control over business logic and data access. This approach offers maximum flexibility but requires careful implementation to avoid security gaps.

**Infrastructure-level authorization** is enforced by network devices, load balancers, or API gateways before requests reach applications. This provides a security perimeter and can block unauthorized requests early, but may lack the context needed for fine-grained decisions.

**Database-level authorization** controls access to data at the storage layer through database permissions and row-level security. This provides a final layer of protection but may not align well with application-level business rules.

## The Relationship Between Authentication and Authorization

### Sequential Dependency

Authentication must precede authorization in the security flow. You cannot determine what someone is allowed to do until you know who they are. However, the strength of authentication affects the trust level for authorization decisions. Stronger authentication methods might grant access to more sensitive resources or operations.

Some systems implement step-up authentication, where certain high-privilege operations require additional authentication even within an existing session. This allows for graduated access based on authentication strength and recency.

### Token-Based Integration

Modern systems often use tokens to bridge authentication and authorization. After successful authentication, the system issues a token that contains identity information and potentially authorization claims. This token is then presented with each request, allowing the system to make both authentication (is this token valid?) and authorization (what does this token allow?) decisions.

Tokens can be opaque references that require server-side lookup, or self-contained tokens like JWTs that include encoded claims. The choice affects performance, scalability, and security characteristics of the system.

## Real-World Applications

### Enterprise Applications

Large organizations typically implement comprehensive identity and access management (IAM) systems that handle both authentication and authorization across multiple applications. Employees authenticate once through SSO and receive access to various systems based on their roles and departments.

These systems often integrate with directory services like Active Directory, implement approval workflows for access requests, and provide detailed audit trails for compliance requirements. They must handle complex scenarios like temporary access for contractors, emergency access procedures, and automated access provisioning based on HR systems.

### Cloud Platforms

Cloud providers like AWS, Azure, and Google Cloud implement sophisticated IAM systems that control access to cloud resources. Users and services authenticate using various methods (passwords, API keys, certificates, federated identity), and authorization is controlled through policies that specify which resources can be accessed and what operations are permitted.

These systems must handle massive scale, complex resource hierarchies, and integration with customer identity systems while providing fine-grained control and comprehensive audit capabilities.

### API Security

APIs use authentication to verify client identity (through API keys, OAuth tokens, or certificates) and authorization to control which endpoints and data each client can access. Rate limiting, quota management, and usage analytics are often integrated with the authorization system.

Modern API security often implements OAuth 2.0 flows that separate authentication (handled by identity providers) from authorization (controlled by resource servers), allowing for flexible integration patterns and third-party access scenarios.

### Microservices Architecture

Distributed microservices require careful coordination of authentication and authorization across service boundaries. Common patterns include using API gateways for authentication and passing identity context between services, implementing service-to-service authentication through certificates or service accounts, and using distributed authorization systems that can make consistent decisions across all services.

## Simple Authentication and Authorization Implementation

```python
import hashlib
import hmac
import time
import jwt
from typing import Dict, List, Optional, Set
from enum import Enum
from dataclasses import dataclass
from datetime import datetime, timedelta

class Role(Enum):
    ADMIN = "admin"
    USER = "user"
    GUEST = "guest"

@dataclass
class User:
    user_id: str
    username: str
    password_hash: str
    roles: Set[Role]
    email: str
    created_at: datetime
    last_login: Optional[datetime] = None
    is_active: bool = True

@dataclass
class Permission:
    resource: str
    action: str
    
    def __str__(self):
        return f"{self.action}:{self.resource}"

@dataclass
class AuthToken:
    user_id: str
    username: str
    roles: List[str]
    issued_at: datetime
    expires_at: datetime

class AuthenticationService:
    def __init__(self, secret_key: str):
        self.secret_key = secret_key
        self.users: Dict[str, User] = {}
        self.sessions: Dict[str, AuthToken] = {}
        self.failed_attempts: Dict[str, List[datetime]] = {}
        self.max_failed_attempts = 5
        self.lockout_duration = timedelta(minutes=15)
    
    def _hash_password(self, password: str, salt: str = None) -> str:
        """Hash password with salt"""
        if salt is None:
            salt = hashlib.sha256(str(time.time()).encode()).hexdigest()[:16]
        
        password_hash = hashlib.pbkdf2_hmac('sha256', 
                                          password.encode(), 
                                          salt.encode(), 
                                          100000)
        return f"{salt}:{password_hash.hex()}"
    
    def _verify_password(self, password: str, password_hash: str) -> bool:
        """Verify password against hash"""
        try:
            salt, hash_hex = password_hash.split(':')
            expected_hash = hashlib.pbkdf2_hmac('sha256',
                                              password.encode(),
                                              salt.encode(),
                                              100000)
            return hmac.compare_digest(expected_hash.hex(), hash_hex)
        except ValueError:
            return False
    
    def _is_account_locked(self, username: str) -> bool:
        """Check if account is locked due to failed attempts"""
        if username not in self.failed_attempts:
            return False
        
        recent_failures = [
            attempt for attempt in self.failed_attempts[username]
            if datetime.now() - attempt < self.lockout_duration
        ]
        
        return len(recent_failures) >= self.max_failed_attempts
    
    def _record_failed_attempt(self, username: str):
        """Record failed authentication attempt"""
        if username not in self.failed_attempts:
            self.failed_attempts[username] = []
        
        self.failed_attempts[username].append(datetime.now())
        
        # Clean up old attempts
        cutoff = datetime.now() - self.lockout_duration
        self.failed_attempts[username] = [
            attempt for attempt in self.failed_attempts[username]
            if attempt > cutoff
        ]
    
    def register_user(self, username: str, password: str, email: str, 
                     roles: Set[Role] = None) -> bool:
        """Register a new user"""
        if username in self.users:
            return False
        
        if roles is None:
            roles = {Role.USER}
        
        password_hash = self._hash_password(password)
        
        user = User(
            user_id=f"user_{len(self.users) + 1}",
            username=username,
            password_hash=password_hash,
            roles=roles,
            email=email,
            created_at=datetime.now()
        )
        
        self.users[username] = user
        print(f"User {username} registered successfully")
        return True
    
    def authenticate(self, username: str, password: str) -> Optional[str]:
        """Authenticate user and return token"""
        # Check if account is locked
        if self._is_account_locked(username):
            print(f"Authentication failed: Account {username} is locked")
            return None
        
        # Check if user exists
        if username not in self.users:
            self._record_failed_attempt(username)
            print(f"Authentication failed: User {username} not found")
            return None
        
        user = self.users[username]
        
        # Check if user is active
        if not user.is_active:
            print(f"Authentication failed: User {username} is inactive")
            return None
        
        # Verify password
        if not self._verify_password(password, user.password_hash):
            self._record_failed_attempt(username)
            print(f"Authentication failed: Invalid password for {username}")
            return None
        
        # Clear failed attempts on successful login
        if username in self.failed_attempts:
            del self.failed_attempts[username]
        
        # Update last login
        user.last_login = datetime.now()
        
        # Generate token
        token_data = {
            'user_id': user.user_id,
            'username': user.username,
            'roles': [role.value for role in user.roles],
            'iat': datetime.now(),
            'exp': datetime.now() + timedelta(hours=24)
        }
        
        token = jwt.encode(token_data, self.secret_key, algorithm='HS256')
        
        # Store session
        auth_token = AuthToken(
            user_id=user.user_id,
            username=user.username,
            roles=[role.value for role in user.roles],
            issued_at=datetime.now(),
            expires_at=datetime.now() + timedelta(hours=24)
        )
        
        self.sessions[token] = auth_token
        
        print(f"User {username} authenticated successfully")
        return token
    
    def validate_token(self, token: str) -> Optional[AuthToken]:
        """Validate authentication token"""
        try:
            # Decode JWT
            payload = jwt.decode(token, self.secret_key, algorithms=['HS256'])
            
            # Check if session exists
            if token not in self.sessions:
                return None
            
            auth_token = self.sessions[token]
            
            # Check if token is expired
            if datetime.now() > auth_token.expires_at:
                del self.sessions[token]
                return None
            
            return auth_token
            
        except jwt.InvalidTokenError:
            return None
    
    def logout(self, token: str) -> bool:
        """Logout user by invalidating token"""
        if token in self.sessions:
            username = self.sessions[token].username
            del self.sessions[token]
            print(f"User {username} logged out")
            return True
        return False

class AuthorizationService:
    def __init__(self):
        # Define role-based permissions
        self.role_permissions: Dict[Role, Set[Permission]] = {
            Role.ADMIN: {
                Permission("users", "create"),
                Permission("users", "read"),
                Permission("users", "update"),
                Permission("users", "delete"),
                Permission("posts", "create"),
                Permission("posts", "read"),
                Permission("posts", "update"),
                Permission("posts", "delete"),
                Permission("system", "configure"),
            },
            Role.USER: {
                Permission("posts", "create"),
                Permission("posts", "read"),
                Permission("posts", "update_own"),
                Permission("posts", "delete_own"),
                Permission("profile", "read"),
                Permission("profile", "update"),
            },
            Role.GUEST: {
                Permission("posts", "read"),
            }
        }
        
        # Resource ownership tracking
        self.resource_owners: Dict[str, str] = {}
    
    def has_permission(self, auth_token: AuthToken, resource: str, action: str, 
                      resource_id: str = None) -> bool:
        """Check if user has permission for specific action on resource"""
        user_permissions = set()
        
        # Collect permissions from all user roles
        for role_str in auth_token.roles:
            try:
                role = Role(role_str)
                if role in self.role_permissions:
                    user_permissions.update(self.role_permissions[role])
            except ValueError:
                continue
        
        # Check for exact permission match
        required_permission = Permission(resource, action)
        if required_permission in user_permissions:
            return True
        
        # Check for ownership-based permissions
        if resource_id and action.endswith('_own'):
            base_action = action.replace('_own', '')
            ownership_permission = Permission(resource, base_action + '_own')
            
            if ownership_permission in user_permissions:
                # Check if user owns the resource
                resource_key = f"{resource}:{resource_id}"
                return self.resource_owners.get(resource_key) == auth_token.user_id
        
        return False
    
    def set_resource_owner(self, resource: str, resource_id: str, owner_user_id: str):
        """Set ownership of a resource"""
        resource_key = f"{resource}:{resource_id}"
        self.resource_owners[resource_key] = owner_user_id
    
    def get_user_permissions(self, auth_token: AuthToken) -> Set[Permission]:
        """Get all permissions for a user"""
        user_permissions = set()
        
        for role_str in auth_token.roles:
            try:
                role = Role(role_str)
                if role in self.role_permissions:
                    user_permissions.update(self.role_permissions[role])
            except ValueError:
                continue
        
        return user_permissions

# Demonstration of authentication and authorization
def demo_auth_system():
    print("=== Authentication & Authorization Demo ===")
    
    # Initialize services
    auth_service = AuthenticationService("secret_key_123")
    authz_service = AuthorizationService()
    
    # Register users
    print("\n=== User Registration ===")
    auth_service.register_user("admin", "admin_pass", "admin@example.com", {Role.ADMIN})
    auth_service.register_user("john", "john_pass", "john@example.com", {Role.USER})
    auth_service.register_user("guest", "guest_pass", "guest@example.com", {Role.GUEST})
    
    # Test authentication
    print("\n=== Authentication Tests ===")
    
    # Successful authentication
    admin_token = auth_service.authenticate("admin", "admin_pass")
    john_token = auth_service.authenticate("john", "john_pass")
    guest_token = auth_service.authenticate("guest", "guest_pass")
    
    # Failed authentication
    failed_token = auth_service.authenticate("john", "wrong_pass")
    print(f"Failed auth result: {failed_token}")
    
    # Test authorization
    print("\n=== Authorization Tests ===")
    
    if admin_token:
        admin_auth = auth_service.validate_token(admin_token)
        if admin_auth:
            print(f"Admin permissions:")
            permissions = authz_service.get_user_permissions(admin_auth)
            for perm in sorted(permissions, key=str):
                print(f"  - {perm}")
            
            # Test admin permissions
            can_delete_users = authz_service.has_permission(admin_auth, "users", "delete")
            print(f"Admin can delete users: {can_delete_users}")
    
    if john_token:
        john_auth = auth_service.validate_token(john_token)
        if john_auth:
            print(f"\nJohn permissions:")
            permissions = authz_service.get_user_permissions(john_auth)
            for perm in sorted(permissions, key=str):
                print(f"  - {perm}")
            
            # Test user permissions
            can_read_posts = authz_service.has_permission(john_auth, "posts", "read")
            can_delete_users = authz_service.has_permission(john_auth, "users", "delete")
            print(f"John can read posts: {can_read_posts}")
            print(f"John can delete users: {can_delete_users}")
            
            # Test ownership-based permissions
            authz_service.set_resource_owner("posts", "123", john_auth.user_id)
            can_update_own_post = authz_service.has_permission(john_auth, "posts", "update_own", "123")
            can_update_other_post = authz_service.has_permission(john_auth, "posts", "update_own", "456")
            print(f"John can update own post: {can_update_own_post}")
            print(f"John can update other's post: {can_update_other_post}")
    
    # Test token validation and logout
    print("\n=== Session Management ===")
    if john_token:
        valid_token = auth_service.validate_token(john_token)
        print(f"Token valid before logout: {valid_token is not None}")
        
        auth_service.logout(john_token)
        
        valid_token = auth_service.validate_token(john_token)
        print(f"Token valid after logout: {valid_token is not None}")

if __name__ == "__main__":
    demo_auth_system()
```

## Common Interview Questions

**Q: What's the difference between authentication and authorization?**

Authentication verifies identity ("Who are you?") while authorization controls access ("What can you do?"). Authentication comes first - you must prove who you are before the system can determine what you're allowed to access. Authentication methods include passwords, multi-factor authentication, certificates, or biometrics. Authorization uses that verified identity to make access decisions based on roles, permissions, or policies. For example, logging into a banking app is authentication, while being able to view only your own accounts (not others') is authorization. You can't have authorization without authentication, but you can authenticate without being authorized for specific resources.

**Q: What are the different types of authentication methods and their trade-offs?**

Password-based: Simple but vulnerable to attacks, password reuse, and user fatigue. Multi-factor authentication (MFA): Much more secure by combining multiple factors (something you know/have/are) but adds complexity and potential user friction. Single Sign-On (SSO): Improves user experience and can strengthen security through centralization, but creates single points of failure. Certificate-based: Very secure and good for system-to-system auth, but complex to manage and deploy. Biometric: Convenient and secure, but requires special hardware and raises privacy concerns. The choice depends on security requirements, user experience needs, technical constraints, and compliance requirements.

**Q: Explain Role-Based Access Control (RBAC) vs Attribute-Based Access Control (ABAC).**

RBAC assigns permissions to roles, then assigns users to roles. It's simple to understand and manage, works well for hierarchical organizations, and provides good separation of duties. However, it can become rigid and create role explosion in complex scenarios. ABAC makes decisions based on attributes of users, resources, actions, and environment (time, location, etc.). It's much more flexible and can handle complex policies, but is harder to implement, debug, and audit. RBAC is better for stable, well-defined organizational structures, while ABAC suits dynamic environments with complex access requirements. Many modern systems use hybrid approaches combining both models.

**Q: How do you implement secure session management in distributed systems?**

Secure session management in distributed systems involves several strategies: Use stateless tokens (like JWTs) that contain necessary information and can be validated without server-side storage, but ensure proper token expiration and rotation. Implement distributed session stores (Redis, database) that all services can access for stateful sessions. Use secure token transmission (HTTPS only, secure cookies, proper headers). Implement token refresh mechanisms to limit exposure time. Handle token revocation through blacklists or short expiration times. Consider service-to-service authentication using certificates or service accounts. Implement proper logout that invalidates tokens across all services. Monitor for suspicious activity and implement rate limiting to prevent abuse.

## Authentication and Authorization Best Practices

### Implement Defense in Depth

Use multiple layers of security rather than relying on a single authentication or authorization mechanism. Combine strong authentication with fine-grained authorization, implement both application-level and infrastructure-level controls, and use monitoring and anomaly detection to identify potential security issues.

### Follow the Principle of Least Privilege

Grant users and systems only the minimum permissions necessary to perform their required functions. Regularly review and audit permissions, implement time-limited access for temporary needs, and use role hierarchies to manage permissions efficiently while avoiding over-privileging.

### Secure Token Management

Use secure token generation with sufficient entropy, implement proper token expiration and rotation policies, transmit tokens only over secure channels, and store tokens securely on both client and server sides. Consider the trade-offs between stateful and stateless tokens based on your scalability and security requirements.

### Implement Comprehensive Auditing

Log all authentication and authorization events including successful and failed attempts, permission changes, and administrative actions. Ensure logs are tamper-proof, regularly monitored, and retained according to compliance requirements. Use structured logging to enable effective analysis and alerting.

### Plan for Scalability and Performance

Design authentication and authorization systems that can handle your expected load without becoming bottlenecks. Use caching appropriately, implement efficient permission lookup mechanisms, and consider the performance impact of complex authorization policies. Plan for horizontal scaling of identity and access management components.

### Regular Security Reviews

Conduct regular reviews of authentication mechanisms, authorization policies, and access patterns. Test for common vulnerabilities, review user permissions and role assignments, and ensure that security policies align with current business requirements and threat landscape.

Understanding the distinction between authentication and authorization is fundamental to building secure systems. While they work together to protect resources, each serves a specific purpose and requires different implementation strategies and security considerations. Proper implementation of both is essential for maintaining system security while providing appropriate access to legitimate users.
