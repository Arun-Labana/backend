# JWT & OAuth2

## What are JWT and OAuth2?

JWT (JSON Web Tokens) and OAuth2 are complementary technologies that work together to provide secure authentication and authorization in modern web applications and APIs. Think of OAuth2 like a valet parking system at a fancy restaurant - when you arrive, you don't give the valet your house keys or wallet, just your car keys with permission to park your car. Similarly, OAuth2 allows applications to access specific user resources without requiring the user's actual credentials. JWT is like the numbered ticket the valet gives you - it's a secure, self-contained token that proves you have the right to retrieve your car later.

While OAuth2 defines the authorization framework and flows for granting access permissions, JWT provides a standardized format for securely transmitting information between parties as tokens. Together, they enable secure, scalable authentication and authorization systems that support modern application architectures including single-page applications, mobile apps, and microservices.

## Understanding JWT (JSON Web Tokens)

### JWT Structure and Components

A JWT is a compact, URL-safe token that consists of three parts separated by dots: header.payload.signature. The header contains metadata about the token including the signing algorithm used. The payload contains the claims (statements about the user and additional data), and the signature ensures the token hasn't been tampered with and verifies the sender's identity.

Each part is Base64URL encoded, making JWTs easy to transmit in HTTP headers, URL parameters, or POST parameters. The self-contained nature of JWTs means they carry all necessary information within the token itself, eliminating the need for server-side session storage and enabling stateless authentication.

### JWT Claims and Security

JWT claims are statements about an entity (typically the user) and additional data. Standard claims include "iss" (issuer), "exp" (expiration time), "sub" (subject), and "aud" (audience). Custom claims can include user roles, permissions, or any other relevant information needed for authorization decisions.

The signature ensures JWT integrity and authenticity. When using symmetric algorithms like HMAC SHA256, the same secret key is used for both signing and verification. Asymmetric algorithms like RSA or ECDSA use a private key for signing and a public key for verification, enabling distributed verification without sharing secrets.

### JWT Advantages and Limitations

JWTs provide several advantages including stateless authentication (no server-side session storage required), scalability (tokens can be verified independently by any service), and cross-domain support (tokens work across different domains and services). They're particularly well-suited for distributed systems and microservices architectures.

However, JWTs also have limitations. Once issued, they cannot be easily revoked before expiration, making token compromise a concern. They can become large if they contain extensive claims, impacting performance. Token expiration must be carefully balanced between security (shorter expiration) and user experience (longer expiration to avoid frequent re-authentication).

## Understanding OAuth2

### OAuth2 Roles and Flow

OAuth2 defines four roles in the authorization process. The Resource Owner (typically the user) owns the data being accessed. The Client (the application) wants to access the user's data. The Authorization Server handles authentication and issues access tokens. The Resource Server hosts the protected resources and accepts access tokens.

The basic OAuth2 flow involves the client redirecting the user to the authorization server, the user authenticating and granting permission, the authorization server redirecting back to the client with an authorization code, and the client exchanging the code for an access token that can be used to access protected resources.

### OAuth2 Grant Types

**Authorization Code Grant** is the most secure and commonly used flow for web applications. The client receives an authorization code that must be exchanged for tokens on the server side, keeping tokens away from the browser and potential attackers. This flow supports refresh tokens for long-term access.

**Implicit Grant** was designed for public clients like single-page applications where client secrets cannot be securely stored. However, it's less secure because tokens are exposed in the browser and URL fragments. Modern best practices recommend using Authorization Code with PKCE instead.

**Client Credentials Grant** is used for server-to-server authentication where the client acts on its own behalf rather than on behalf of a user. This is common for API integrations and background services that need to access resources without user interaction.

**Resource Owner Password Credentials Grant** allows the client to collect the user's username and password directly. This grant type should only be used when there's high trust between the user and client, and other flows aren't feasible.

### PKCE (Proof Key for Code Exchange)

PKCE enhances the security of OAuth2 flows, particularly for public clients like mobile apps and single-page applications. It works by having the client generate a random code verifier and its corresponding code challenge. The code challenge is sent with the authorization request, and the code verifier is sent when exchanging the authorization code for tokens.

This prevents authorization code interception attacks because an attacker who intercepts the authorization code cannot exchange it for tokens without the original code verifier. PKCE is now recommended for all OAuth2 clients, not just public ones.

## JWT and OAuth2 Integration

### Access Tokens as JWTs

While OAuth2 doesn't specify the format of access tokens, JWTs are commonly used because they're self-contained and can carry authorization information. When an OAuth2 authorization server issues JWT access tokens, resource servers can validate and extract user information directly from the token without calling back to the authorization server.

This approach improves performance and scalability but requires careful consideration of token size, expiration times, and revocation strategies. JWT access tokens should have short expiration times to limit the impact of token compromise.

### Refresh Token Strategy

Refresh tokens provide a way to obtain new access tokens without requiring user re-authentication. They're typically longer-lived than access tokens and are used to maintain user sessions while keeping access tokens short-lived for security.

When using JWTs as access tokens, refresh tokens become crucial for managing token lifecycle. The client can use a refresh token to obtain a new JWT access token when the current one expires, providing a balance between security and user experience.

### Token Validation and Claims

Resource servers must validate JWT tokens by verifying the signature, checking expiration times, and validating claims like issuer and audience. The token's claims can then be used to make authorization decisions without additional database lookups.

This distributed validation capability is particularly valuable in microservices architectures where multiple services need to make authorization decisions independently while maintaining consistent security policies.

## Real-World Applications

### Single Sign-On (SSO) Systems

Enterprise SSO systems use OAuth2 flows to enable users to authenticate once and access multiple applications. When a user logs into the corporate portal, they receive JWT tokens that can be used to access various internal applications without additional authentication.

The JWT tokens contain user identity information and role assignments that applications can use to customize the user experience and enforce authorization policies. This approach reduces password fatigue and improves security by centralizing authentication.

### API Gateway Authentication

API gateways often use JWT tokens issued through OAuth2 flows to authenticate and authorize API requests. Clients obtain JWT tokens from an OAuth2 authorization server and include them in API requests. The gateway validates the tokens and extracts user information for rate limiting, logging, and routing decisions.

This pattern enables fine-grained API access control while maintaining high performance through stateless token validation. Different client applications can have different scopes and permissions encoded in their JWT tokens.

### Mobile Application Security

Mobile applications use OAuth2 with PKCE to securely authenticate users and access backend APIs. The mobile app redirects users to the authorization server through the system browser, receives an authorization code, and exchanges it for JWT tokens using PKCE for additional security.

The JWT tokens are stored securely on the device and included in API requests. Short-lived access tokens with refresh token rotation provide a good balance between security and user experience for mobile applications.

### Third-Party Integrations

OAuth2 enables secure third-party integrations where external applications can access user data with explicit permission. For example, a photo printing service can use OAuth2 to access a user's photos from a cloud storage provider without requiring the user's storage credentials.

The OAuth2 flow ensures users maintain control over their data by explicitly granting permissions, and JWT tokens can include scope limitations that restrict what data the third-party application can access.

## Simple JWT and OAuth2 Implementation

```python
import jwt
import time
import secrets
import hashlib
import base64
from typing import Dict, List, Optional, Set
from datetime import datetime, timedelta
from dataclasses import dataclass
from enum import Enum
import urllib.parse

class GrantType(Enum):
    AUTHORIZATION_CODE = "authorization_code"
    CLIENT_CREDENTIALS = "client_credentials"
    REFRESH_TOKEN = "refresh_token"

@dataclass
class OAuthClient:
    client_id: str
    client_secret: str
    redirect_uris: List[str]
    scopes: Set[str]
    grant_types: Set[GrantType]
    is_public: bool = False

@dataclass
class AuthorizationCode:
    code: str
    client_id: str
    user_id: str
    scopes: Set[str]
    redirect_uri: str
    expires_at: datetime
    code_challenge: Optional[str] = None
    code_challenge_method: Optional[str] = None

@dataclass
class AccessToken:
    token: str
    client_id: str
    user_id: Optional[str]
    scopes: Set[str]
    expires_at: datetime

@dataclass
class RefreshToken:
    token: str
    client_id: str
    user_id: Optional[str]
    scopes: Set[str]
    expires_at: datetime

class JWTService:
    def __init__(self, secret_key: str, issuer: str = "auth-server"):
        self.secret_key = secret_key
        self.issuer = issuer
        self.algorithm = "HS256"
    
    def create_access_token(self, user_id: str, client_id: str, scopes: Set[str], 
                           expires_in: int = 3600) -> str:
        """Create a JWT access token"""
        now = datetime.utcnow()
        payload = {
            'iss': self.issuer,
            'sub': user_id,
            'aud': client_id,
            'iat': now,
            'exp': now + timedelta(seconds=expires_in),
            'scope': ' '.join(scopes),
            'token_type': 'access_token'
        }
        
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def create_refresh_token(self, user_id: str, client_id: str, scopes: Set[str],
                           expires_in: int = 86400 * 30) -> str:  # 30 days
        """Create a JWT refresh token"""
        now = datetime.utcnow()
        payload = {
            'iss': self.issuer,
            'sub': user_id,
            'aud': client_id,
            'iat': now,
            'exp': now + timedelta(seconds=expires_in),
            'scope': ' '.join(scopes),
            'token_type': 'refresh_token'
        }
        
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def validate_token(self, token: str) -> Optional[Dict]:
        """Validate and decode a JWT token"""
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            
            # Check if token is expired
            if datetime.utcnow() > datetime.fromtimestamp(payload['exp']):
                return None
            
            return payload
        except jwt.InvalidTokenError:
            return None
    
    def extract_scopes(self, token: str) -> Set[str]:
        """Extract scopes from a JWT token"""
        payload = self.validate_token(token)
        if payload and 'scope' in payload:
            return set(payload['scope'].split())
        return set()

class OAuth2Server:
    def __init__(self, jwt_service: JWTService):
        self.jwt_service = jwt_service
        self.clients: Dict[str, OAuthClient] = {}
        self.authorization_codes: Dict[str, AuthorizationCode] = {}
        self.users: Dict[str, Dict] = {
            'user1': {'username': 'john', 'password': 'password123'},
            'user2': {'username': 'jane', 'password': 'password456'}
        }
    
    def register_client(self, client: OAuthClient):
        """Register an OAuth2 client"""
        self.clients[client.client_id] = client
        print(f"Registered OAuth2 client: {client.client_id}")
    
    def _generate_code_challenge(self, code_verifier: str) -> str:
        """Generate PKCE code challenge from verifier"""
        digest = hashlib.sha256(code_verifier.encode()).digest()
        return base64.urlsafe_b64encode(digest).decode().rstrip('=')
    
    def authorize(self, client_id: str, redirect_uri: str, scopes: Set[str],
                 user_id: str, code_challenge: str = None, 
                 code_challenge_method: str = None) -> str:
        """Handle authorization request and return authorization code"""
        
        # Validate client
        if client_id not in self.clients:
            raise ValueError("Invalid client_id")
        
        client = self.clients[client_id]
        
        # Validate redirect URI
        if redirect_uri not in client.redirect_uris:
            raise ValueError("Invalid redirect_uri")
        
        # Validate scopes
        if not scopes.issubset(client.scopes):
            raise ValueError("Invalid scopes")
        
        # Generate authorization code
        code = secrets.token_urlsafe(32)
        
        auth_code = AuthorizationCode(
            code=code,
            client_id=client_id,
            user_id=user_id,
            scopes=scopes,
            redirect_uri=redirect_uri,
            expires_at=datetime.utcnow() + timedelta(minutes=10),
            code_challenge=code_challenge,
            code_challenge_method=code_challenge_method
        )
        
        self.authorization_codes[code] = auth_code
        
        print(f"Generated authorization code for user {user_id}, client {client_id}")
        return code
    
    def exchange_code_for_tokens(self, client_id: str, client_secret: str, 
                               code: str, redirect_uri: str, 
                               code_verifier: str = None) -> Dict:
        """Exchange authorization code for access and refresh tokens"""
        
        # Validate client credentials
        if client_id not in self.clients:
            raise ValueError("Invalid client")
        
        client = self.clients[client_id]
        
        if not client.is_public and client.client_secret != client_secret:
            raise ValueError("Invalid client credentials")
        
        # Validate authorization code
        if code not in self.authorization_codes:
            raise ValueError("Invalid authorization code")
        
        auth_code = self.authorization_codes[code]
        
        # Check expiration
        if datetime.utcnow() > auth_code.expires_at:
            del self.authorization_codes[code]
            raise ValueError("Authorization code expired")
        
        # Validate client and redirect URI
        if auth_code.client_id != client_id or auth_code.redirect_uri != redirect_uri:
            raise ValueError("Invalid authorization code")
        
        # Validate PKCE if present
        if auth_code.code_challenge:
            if not code_verifier:
                raise ValueError("Code verifier required")
            
            if auth_code.code_challenge_method == "S256":
                expected_challenge = self._generate_code_challenge(code_verifier)
                if auth_code.code_challenge != expected_challenge:
                    raise ValueError("Invalid code verifier")
        
        # Generate tokens
        access_token = self.jwt_service.create_access_token(
            auth_code.user_id, client_id, auth_code.scopes
        )
        
        refresh_token = self.jwt_service.create_refresh_token(
            auth_code.user_id, client_id, auth_code.scopes
        )
        
        # Clean up authorization code
        del self.authorization_codes[code]
        
        print(f"Issued tokens for user {auth_code.user_id}, client {client_id}")
        
        return {
            'access_token': access_token,
            'refresh_token': refresh_token,
            'token_type': 'Bearer',
            'expires_in': 3600,
            'scope': ' '.join(auth_code.scopes)
        }
    
    def refresh_access_token(self, client_id: str, client_secret: str, 
                           refresh_token: str) -> Dict:
        """Refresh access token using refresh token"""
        
        # Validate client
        if client_id not in self.clients:
            raise ValueError("Invalid client")
        
        client = self.clients[client_id]
        
        if not client.is_public and client.client_secret != client_secret:
            raise ValueError("Invalid client credentials")
        
        # Validate refresh token
        payload = self.jwt_service.validate_token(refresh_token)
        if not payload or payload.get('token_type') != 'refresh_token':
            raise ValueError("Invalid refresh token")
        
        if payload['aud'] != client_id:
            raise ValueError("Token not issued for this client")
        
        # Generate new access token
        scopes = set(payload['scope'].split())
        access_token = self.jwt_service.create_access_token(
            payload['sub'], client_id, scopes
        )
        
        print(f"Refreshed access token for user {payload['sub']}, client {client_id}")
        
        return {
            'access_token': access_token,
            'token_type': 'Bearer',
            'expires_in': 3600,
            'scope': payload['scope']
        }
    
    def client_credentials_grant(self, client_id: str, client_secret: str, 
                               scopes: Set[str]) -> Dict:
        """Handle client credentials grant for machine-to-machine auth"""
        
        # Validate client
        if client_id not in self.clients:
            raise ValueError("Invalid client")
        
        client = self.clients[client_id]
        
        if client.client_secret != client_secret:
            raise ValueError("Invalid client credentials")
        
        if GrantType.CLIENT_CREDENTIALS not in client.grant_types:
            raise ValueError("Client credentials grant not allowed")
        
        # Validate scopes
        if not scopes.issubset(client.scopes):
            raise ValueError("Invalid scopes")
        
        # Generate access token (no user context)
        access_token = self.jwt_service.create_access_token(
            client_id, client_id, scopes  # Use client_id as subject for machine auth
        )
        
        print(f"Issued client credentials token for client {client_id}")
        
        return {
            'access_token': access_token,
            'token_type': 'Bearer',
            'expires_in': 3600,
            'scope': ' '.join(scopes)
        }

# Resource server for validating tokens and serving protected resources
class ResourceServer:
    def __init__(self, jwt_service: JWTService):
        self.jwt_service = jwt_service
    
    def validate_request(self, authorization_header: str, required_scope: str = None) -> Dict:
        """Validate incoming request with JWT token"""
        
        if not authorization_header or not authorization_header.startswith('Bearer '):
            raise ValueError("Missing or invalid authorization header")
        
        token = authorization_header[7:]  # Remove 'Bearer ' prefix
        
        # Validate token
        payload = self.jwt_service.validate_token(token)
        if not payload:
            raise ValueError("Invalid or expired token")
        
        # Check required scope
        if required_scope:
            token_scopes = set(payload.get('scope', '').split())
            if required_scope not in token_scopes:
                raise ValueError(f"Insufficient scope. Required: {required_scope}")
        
        return payload
    
    def get_user_profile(self, authorization_header: str) -> Dict:
        """Protected endpoint that requires 'profile' scope"""
        payload = self.validate_request(authorization_header, 'profile')
        
        return {
            'user_id': payload['sub'],
            'client_id': payload['aud'],
            'scopes': payload['scope'].split(),
            'message': 'This is protected user profile data'
        }
    
    def get_admin_data(self, authorization_header: str) -> Dict:
        """Protected endpoint that requires 'admin' scope"""
        payload = self.validate_request(authorization_header, 'admin')
        
        return {
            'user_id': payload['sub'],
            'client_id': payload['aud'],
            'message': 'This is protected admin data'
        }

# Demonstration of JWT and OAuth2 integration
def demo_jwt_oauth2():
    print("=== JWT & OAuth2 Demo ===")
    
    # Initialize services
    jwt_service = JWTService("secret_key_for_jwt_signing")
    oauth_server = OAuth2Server(jwt_service)
    resource_server = ResourceServer(jwt_service)
    
    # Register OAuth2 clients
    print("\n=== Client Registration ===")
    
    web_client = OAuthClient(
        client_id="web_app_123",
        client_secret="web_secret_456",
        redirect_uris=["https://webapp.example.com/callback"],
        scopes={"profile", "email"},
        grant_types={GrantType.AUTHORIZATION_CODE, GrantType.REFRESH_TOKEN}
    )
    
    api_client = OAuthClient(
        client_id="api_service_789",
        client_secret="api_secret_abc",
        redirect_uris=[],
        scopes={"admin", "api_access"},
        grant_types={GrantType.CLIENT_CREDENTIALS}
    )
    
    oauth_server.register_client(web_client)
    oauth_server.register_client(api_client)
    
    # Authorization Code Flow
    print("\n=== Authorization Code Flow ===")
    
    # Step 1: Get authorization code
    auth_code = oauth_server.authorize(
        client_id="web_app_123",
        redirect_uri="https://webapp.example.com/callback",
        scopes={"profile", "email"},
        user_id="user1"
    )
    print(f"Authorization code: {auth_code[:10]}...")
    
    # Step 2: Exchange code for tokens
    tokens = oauth_server.exchange_code_for_tokens(
        client_id="web_app_123",
        client_secret="web_secret_456",
        code=auth_code,
        redirect_uri="https://webapp.example.com/callback"
    )
    
    print(f"Access token issued: {tokens['access_token'][:50]}...")
    print(f"Refresh token issued: {tokens['refresh_token'][:50]}...")
    
    # Step 3: Use access token to access protected resource
    print("\n=== Accessing Protected Resources ===")
    
    auth_header = f"Bearer {tokens['access_token']}"
    
    try:
        profile_data = resource_server.get_user_profile(auth_header)
        print(f"Profile data: {profile_data}")
    except ValueError as e:
        print(f"Access denied: {e}")
    
    try:
        admin_data = resource_server.get_admin_data(auth_header)
        print(f"Admin data: {admin_data}")
    except ValueError as e:
        print(f"Access denied: {e}")
    
    # Step 4: Refresh access token
    print("\n=== Token Refresh ===")
    
    new_tokens = oauth_server.refresh_access_token(
        client_id="web_app_123",
        client_secret="web_secret_456",
        refresh_token=tokens['refresh_token']
    )
    
    print(f"New access token: {new_tokens['access_token'][:50]}...")
    
    # Client Credentials Flow
    print("\n=== Client Credentials Flow ===")
    
    client_tokens = oauth_server.client_credentials_grant(
        client_id="api_service_789",
        client_secret="api_secret_abc",
        scopes={"admin", "api_access"}
    )
    
    print(f"Client credentials token: {client_tokens['access_token'][:50]}...")
    
    # Use client credentials token
    client_auth_header = f"Bearer {client_tokens['access_token']}"
    
    try:
        admin_data = resource_server.get_admin_data(client_auth_header)
        print(f"Admin data (client credentials): {admin_data}")
    except ValueError as e:
        print(f"Access denied: {e}")

if __name__ == "__main__":
    demo_jwt_oauth2()
```

## Common Interview Questions

**Q: What's the difference between JWT and OAuth2, and how do they work together?**

OAuth2 is an authorization framework that defines how applications can securely access user resources, while JWT is a token format for securely transmitting information. OAuth2 defines the flows and protocols for obtaining access tokens, while JWT defines how those tokens can be structured and validated. They work together when OAuth2 authorization servers issue JWT-formatted access tokens, enabling stateless authentication where resource servers can validate tokens locally without calling back to the authorization server. OAuth2 handles the "how to get permission" while JWT handles the "how to prove permission."

**Q: What are the main OAuth2 grant types and when would you use each?**

Authorization Code Grant: Most secure, used for web applications with server-side components. Supports refresh tokens and keeps tokens away from browsers. Implicit Grant: Originally for SPAs, now deprecated in favor of Authorization Code with PKCE due to security concerns. Client Credentials Grant: For machine-to-machine authentication where the application acts on its own behalf, not a user's. Resource Owner Password Credentials: Direct username/password collection, only for highly trusted applications. PKCE (Proof Key for Code Exchange): Security extension for public clients like mobile apps and SPAs, now recommended for all clients.

**Q: What are the security considerations when using JWTs?**

Key security considerations include: Token storage (secure storage on client side, avoid localStorage for sensitive tokens), token expiration (balance security vs UX, use short-lived access tokens with refresh tokens), signature verification (always validate signatures and standard claims like exp, iss, aud), secret management (protect signing keys, use strong algorithms), token revocation (JWTs can't be easily revoked, so use short expiration times), and information disclosure (don't put sensitive data in JWT payload as it's only base64 encoded). Consider using opaque tokens for highly sensitive applications where revocation is critical.

**Q: How do you handle token refresh and revocation in JWT-based systems?**

Token refresh uses refresh tokens to obtain new access tokens without user re-authentication. Implement short-lived access tokens (15-60 minutes) with longer-lived refresh tokens (days/weeks). Use refresh token rotation where each refresh operation issues a new refresh token and invalidates the old one. For revocation, maintain a blacklist of revoked tokens (though this reduces statelessness benefits), use short expiration times to limit exposure, implement logout endpoints that invalidate refresh tokens, and consider using opaque tokens for critical operations that require immediate revocation. Monitor for suspicious token usage patterns and implement automatic revocation triggers.

## JWT and OAuth2 Best Practices

### Implement Proper Token Lifecycle Management

Use short-lived access tokens (15-60 minutes) combined with longer-lived refresh tokens to balance security and user experience. Implement refresh token rotation where each refresh operation issues new tokens and invalidates old ones. Store refresh tokens securely and implement proper logout procedures that invalidate all tokens.

### Secure Token Storage and Transmission

Store tokens securely on the client side using appropriate mechanisms for each platform (secure HTTP-only cookies for web, keychain/keystore for mobile). Always transmit tokens over HTTPS and include proper security headers. Avoid storing sensitive information in JWT payloads since they're only base64 encoded, not encrypted.

### Validate All Token Claims

Always validate JWT signatures, expiration times, issuer, audience, and other standard claims. Implement proper error handling for invalid or expired tokens. Use strong signing algorithms (RS256 or ES256 for asymmetric, HS256 minimum for symmetric) and protect signing keys appropriately.

### Implement Comprehensive Scope Management

Design granular scopes that follow the principle of least privilege. Validate scopes at both the authorization server and resource server levels. Document scope meanings clearly and implement scope inheritance hierarchies where appropriate. Monitor scope usage to identify potential security issues or over-privileging.

### Plan for Scalability and Performance

Design token validation to be efficient and scalable. Cache public keys for JWT validation, implement proper token introspection endpoints for opaque tokens, and consider the performance impact of token size. Use appropriate caching strategies for authorization decisions while maintaining security.

### Monitor and Audit Token Usage

Implement comprehensive logging for token issuance, validation, and usage. Monitor for suspicious patterns like token reuse, unusual scope requests, or high refresh rates. Set up alerting for security events and maintain audit trails for compliance requirements.

JWT and OAuth2 together provide a powerful foundation for modern authentication and authorization systems. Understanding their proper implementation, security considerations, and best practices is essential for building secure, scalable applications that can handle diverse client types and integration scenarios while maintaining user privacy and system security.
