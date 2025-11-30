# Authentication Architecture

This document details WiWebb's authentication and authorization system, including token-based auth, role-based access control, and security measures.

## Authentication Overview

WiWebb uses **token-based authentication** powered by `dj-rest-auth` and `django-allauth`, providing secure, stateless authentication for both web and mobile clients.

```mermaid
graph TB
    Client[Client Application] --> Login[Login Request<br/>POST /auth/login/]
    Login --> Django[Django Backend]
    Django --> Validate[Validate Credentials]
    Validate -->|Valid| CreateToken[Create Auth Token]
    Validate -->|Invalid| Reject[401 Unauthorized]
    CreateToken --> StoreToken[Store Token in DB]
    StoreToken --> ReturnToken[Return Token to Client]
    ReturnToken --> Client
    Client --> UseToken[Subsequent Requests<br/>Authorization: Token xxx]
    UseToken --> VerifyToken[Verify Token]
    VerifyToken -->|Valid| GetUser[Get User from Token]
    VerifyToken -->|Invalid| Reject2[401 Unauthorized]
    GetUser --> CheckPerms[Check Permissions]
    CheckPerms -->|Authorized| Access[Access Resource]
    CheckPerms -->|Forbidden| Reject3[403 Forbidden]

    style CreateToken fill:#e8f5e9
    style VerifyToken fill:#e3f2fd
    style CheckPerms fill:#fff3e0
    style Access fill:#c8e6c9
    style Reject fill:#ffcdd2
    style Reject2 fill:#ffcdd2
    style Reject3 fill:#ffcdd2
```

## Authentication Flow

### Login Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant Auth
    participant DB
    participant Redis

    User->>Frontend: Enter credentials
    Frontend->>API: POST /auth/login/<br/>{username, password}

    API->>Auth: Authenticate
    Auth->>DB: SELECT * FROM users<br/>WHERE username=?
    DB-->>Auth: User record

    Auth->>Auth: Verify password hash
    alt Password Valid
        Auth->>DB: Get or create auth token
        DB-->>Auth: Token object
        Auth->>Redis: Cache user data
        Auth-->>API: {key: "token123...", user: {...}}
        API-->>Frontend: 200 OK + Token
        Frontend->>Frontend: Store token (localStorage)
        Frontend->>API: GET /auth/user/<br/>Authorization: Token token123
        API->>Redis: Check cache
        alt Cache Hit
            Redis-->>API: User data
        else Cache Miss
            API->>DB: Get user details
            DB-->>API: User object
            API->>Redis: Cache user data
        end
        API-->>Frontend: User profile
        Frontend-->>User: Redirect to dashboard
    else Password Invalid
        Auth-->>API: Invalid credentials
        API-->>Frontend: 400 Bad Request
        Frontend-->>User: Show error
    end
```

### Logout Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant DB
    participant Redis

    User->>Frontend: Click logout
    Frontend->>API: POST /auth/logout/<br/>Authorization: Token token123
    API->>DB: DELETE token WHERE key=token123
    DB-->>API: Token deleted
    API->>Redis: Clear user cache
    API-->>Frontend: 200 OK
    Frontend->>Frontend: Remove token from storage
    Frontend-->>User: Redirect to login
```

### Registration Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant Validation
    participant DB

    User->>Frontend: Fill registration form
    Frontend->>API: POST /auth/registration/<br/>{username, email, password1, password2}
    API->>Validation: Validate input

    alt Validation Fails
        Validation-->>API: Errors
        API-->>Frontend: 400 Bad Request<br/>{errors: [...]}
        Frontend-->>User: Show validation errors
    else Validation Passes
        Validation->>DB: Check username/email unique
        alt Not Unique
            DB-->>API: Conflict
            API-->>Frontend: 400 Bad Request
            Frontend-->>User: Username/email taken
        else Unique
            API->>DB: INSERT INTO users
            DB-->>API: User created
            API->>DB: Create auth token
            DB-->>API: Token created
            API-->>Frontend: 201 Created<br/>{key: "token123", user: {...}}
            Frontend->>Frontend: Store token
            Frontend-->>User: Redirect to onboarding
        end
    end
```

## Token Management

### Token Model

WiWebb uses Django REST Framework's token model:

```python
class Token(models.Model):
    """
    Authentication token for API access.
    One token per user (single device/session).
    """
    key = models.CharField(max_length=40, primary_key=True)
    user = models.OneToOneField(
        User,
        related_name='auth_token',
        on_delete=models.CASCADE
    )
    created = models.DateTimeField(auto_now_add=True)
```

### Token Generation

```python
import binascii
import os

def generate_key():
    """Generate a random token key."""
    return binascii.hexlify(os.urandom(20)).decode()

# Example token: "9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
```

### Token Storage

**Client-Side Storage:**

=== "Web (localStorage)"
    ```javascript
    // Store token after login
    const { key } = await loginResponse.json();
    localStorage.setItem('auth_token', key);

    // Use token in requests
    const token = localStorage.getItem('auth_token');
    fetch('/api/v1/users/', {
      headers: {
        'Authorization': `Token ${token}`
      }
    });

    // Remove on logout
    localStorage.removeItem('auth_token');
    ```

=== "Mobile (SecureStore)"
    ```typescript
    import * as SecureStore from 'expo-secure-store';

    // Store token
    await SecureStore.setItemAsync('auth_token', token);

    // Retrieve token
    const token = await SecureStore.getItemAsync('auth_token');

    // Delete token
    await SecureStore.deleteItemAsync('auth_token');
    ```

**Server-Side Storage:**

Tokens are stored in the database and cached in Redis:

```python
# Database: permanent storage
Token.objects.create(user=user)

# Redis: fast lookup cache
redis_client.setex(
    f"token:{token.key}",
    3600,  # 1 hour TTL
    json.dumps(user.to_dict())
)
```

## Authorization (RBAC)

### Role Hierarchy

```mermaid
graph TD
    SuperAdmin[SuperAdmin<br/>Highest Privilege] --> Admin[Admin<br/>Multi-Tenant Mgmt]
    Admin --> TenantOwner[Tenant Owner<br/>Single Tenant Mgmt]
    TenantOwner --> Subscriber[Subscriber<br/>End User]

    SuperAdmin -->|Can Do| SA_Perms[• Manage all tenants<br/>• Manage all users<br/>• Assign any role<br/>• System configuration]
    Admin -->|Can Do| A_Perms[• Manage all tenants<br/>• Manage all users<br/>• Assign roles except SuperAdmin<br/>• View system stats]
    TenantOwner -->|Can Do| TO_Perms[• Manage own tenant<br/>• Manage tenant users<br/>• Assign Subscriber role<br/>• View tenant stats]
    Subscriber -->|Can Do| S_Perms[• View own profile<br/>• Update own profile<br/>• Use network services<br/>• View own usage]

    style SuperAdmin fill:#ffebee
    style Admin fill:#fff3e0
    style TenantOwner fill:#e8f5e9
    style Subscriber fill:#e3f2fd
```

### Permission Matrix

| Resource | SuperAdmin | Admin | Tenant Owner | Subscriber |
|----------|:----------:|:-----:|:------------:|:----------:|
| **Tenants** |  |  |  |  |
| List all tenants | ✅ | ✅ | ❌ | ❌ |
| View own tenant | ✅ | ✅ | ✅ | ❌ |
| Create tenant | ✅ | ✅ | ❌ | ❌ |
| Update any tenant | ✅ | ✅ | ❌ | ❌ |
| Update own tenant | ✅ | ✅ | ✅ | ❌ |
| Delete tenant | ✅ | ✅ | ❌ | ❌ |
| **Users** |  |  |  |  |
| List all users | ✅ | ✅ | ❌ | ❌ |
| List tenant users | ✅ | ✅ | ✅ | ❌ |
| Create user (any tenant) | ✅ | ✅ | ❌ | ❌ |
| Create user (own tenant) | ✅ | ✅ | ✅ | ❌ |
| Assign SuperAdmin role | ✅ | ❌ | ❌ | ❌ |
| Assign Admin role | ✅ | ✅ | ❌ | ❌ |
| Assign Tenant Owner role | ✅ | ✅ | ❌ | ❌ |
| Assign Subscriber role | ✅ | ✅ | ✅ | ❌ |
| Update any user | ✅ | ✅ | ❌ | ❌ |
| Update tenant user | ✅ | ✅ | ✅ | ❌ |
| Update own profile | ✅ | ✅ | ✅ | ✅ |
| Delete user | ✅ | ✅ | ✅ | ❌ |
| **Subscriptions** |  |  |  |  |
| View all subscriptions | ✅ | ✅ | ❌ | ❌ |
| View tenant subscriptions | ✅ | ✅ | ✅ | ❌ |
| View own subscription | ✅ | ✅ | ✅ | ✅ |
| Create subscription | ✅ | ✅ | ✅ | ✅ |
| Cancel subscription | ✅ | ✅ | ✅ | ✅ |

### Permission Implementation

#### ViewSet-Level Permissions

```python title="api/views.py"
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .permissions import IsSuperAdminOrAdmin, IsTenantOwnerOrAdmin

class TenantViewSet(viewsets.ModelViewSet):
    """Tenant management viewset."""

    serializer_class = TenantSerializer
    permission_classes = [IsAuthenticated]

    def get_permissions(self):
        """Different permissions for different actions."""
        if self.action in ['create', 'destroy']:
            # Only SuperAdmin/Admin can create/delete tenants
            return [IsSuperAdminOrAdmin()]
        elif self.action in ['update', 'partial_update']:
            # TenantOwner can update their own tenant
            return [IsTenantOwnerOrAdmin()]
        return [IsAuthenticated()]

    def get_queryset(self):
        """Filter queryset based on user role."""
        user = self.request.user

        if user.role in ['superadmin', 'admin']:
            return Tenant.objects.all()
        elif user.role == 'tenant_owner' and user.tenant:
            return Tenant.objects.filter(id=user.tenant.id)
        else:
            return Tenant.objects.none()
```

#### Object-Level Permissions

```python title="api/permissions.py"
from rest_framework import permissions

class IsTenantOwnerOrAdmin(permissions.BasePermission):
    """
    Object-level permission to only allow owners of a tenant
    or admins to edit it.
    """

    def has_permission(self, request, view):
        # Authenticated users only
        return request.user and request.user.is_authenticated

    def has_object_permission(self, request, view, obj):
        user = request.user

        # SuperAdmin and Admin have full access
        if user.role in ['superadmin', 'admin']:
            return True

        # Tenant Owner can access their own tenant
        if user.role == 'tenant_owner':
            # Check if object is the user's tenant
            if hasattr(obj, 'id') and obj.id == user.tenant_id:
                return True
            # Check if object belongs to user's tenant
            if hasattr(obj, 'tenant_id') and obj.tenant_id == user.tenant_id:
                return True

        return False
```

### Role Assignment Rules

```mermaid
graph TD
    Request[Role Assignment Request] --> Actor{Who is Actor?}

    Actor -->|SuperAdmin| SA[Can assign ANY role]
    Actor -->|Admin| Admin_Check{Target Role?}
    Actor -->|Tenant Owner| TO_Check{Target Role?}
    Actor -->|Subscriber| Deny1[❌ No permission]

    Admin_Check -->|SuperAdmin| Deny2[❌ Cannot assign SuperAdmin]
    Admin_Check -->|Other| Allow2[✅ Can assign]

    TO_Check -->|Subscriber| Allow3[✅ Can assign]
    TO_Check -->|Other| Deny3[❌ Cannot assign]

    SA --> Validate[Validate Assignment]
    Allow2 --> Validate
    Allow3 --> Validate

    Validate --> TenantCheck{Same Tenant?}
    TenantCheck -->|Yes or N/A| UpdateRole[Update User Role]
    TenantCheck -->|No| Deny4[❌ Cross-tenant assignment]

    UpdateRole --> AuditLog[Create Audit Log]
    AuditLog --> Success[✅ Role Assigned]

    style SA fill:#e8f5e9
    style Allow2 fill:#e8f5e9
    style Allow3 fill:#e8f5e9
    style UpdateRole fill:#e8f5e9
    style Success fill:#c8e6c9
    style Deny1 fill:#ffcdd2
    style Deny2 fill:#ffcdd2
    style Deny3 fill:#ffcdd2
    style Deny4 fill:#ffcdd2
```

**Implementation:**

```python title="api/views.py"
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import status
from .models import AuditLog

class UserViewSet(viewsets.ModelViewSet):

    @action(detail=True, methods=['post'])
    def assign_role(self, request, pk=None):
        """Assign a role to a user."""
        user = self.get_object()
        actor = request.user
        new_role = request.data.get('role')

        # Validate role value
        if new_role not in dict(User.Roles.choices):
            return Response(
                {'error': 'Invalid role'},
                status=status.HTTP_400_BAD_REQUEST
            )

        # Check permissions
        if actor.role == 'subscriber':
            return Response(
                {'error': 'Insufficient permissions'},
                status=status.HTTP_403_FORBIDDEN
            )

        if actor.role == 'tenant_owner':
            # Tenant owners can only assign Subscriber role
            if new_role != User.Roles.SUBSCRIBER:
                return Response(
                    {'error': 'Can only assign Subscriber role'},
                    status=status.HTTP_403_FORBIDDEN
                )
            # Only within their own tenant
            if user.tenant_id != actor.tenant_id:
                return Response(
                    {'error': 'Cannot modify users from other tenants'},
                    status=status.HTTP_403_FORBIDDEN
                )

        if actor.role == 'admin':
            # Admins cannot assign SuperAdmin role
            if new_role == User.Roles.SUPERADMIN:
                return Response(
                    {'error': 'Cannot assign SuperAdmin role'},
                    status=status.HTTP_403_FORBIDDEN
                )

        # SuperAdmin can assign any role (no additional checks)

        # Store old role for audit
        old_role = user.role

        # Update role
        user.role = new_role
        user.save()

        # Create audit log
        AuditLog.objects.create(
            actor=actor,
            target=user,
            action=AuditLog.Actions.ROLE_CHANGED,
            details={
                'old_role': old_role,
                'new_role': new_role
            }
        )

        return Response({
            'message': 'Role updated successfully',
            'user': UserSerializer(user).data
        })
```

## Security Measures

### Password Security

```python
# Password hashing (PBKDF2)
from django.contrib.auth.hashers import make_password, check_password

# Hash password before storing
hashed = make_password('user_password')
# Output: pbkdf2_sha256$260000$...

# Verify password
check_password('user_password', hashed)  # True/False
```

**Password Requirements:**

```python
# settings.py
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {
            'min_length': 8,
        }
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]
```

### Token Security

!!! danger "Token Security Best Practices"
    1. **HTTPS Only**: Always use HTTPS in production
    2. **Token Rotation**: Implement token rotation for long-lived sessions
    3. **Token Expiration**: Set reasonable expiration times
    4. **Secure Storage**: Use secure storage on client (HttpOnly cookies, SecureStore)
    5. **Revocation**: Support immediate token revocation on logout

### CORS Configuration

```python
# settings.py
CORS_ALLOWED_ORIGINS = [
    "http://localhost:5173",  # Frontend dev server
    "https://app.wiwebb.com",  # Production frontend
]

CORS_ALLOW_CREDENTIALS = True

CORS_ALLOW_METHODS = [
    'DELETE',
    'GET',
    'OPTIONS',
    'PATCH',
    'POST',
    'PUT',
]

CORS_ALLOW_HEADERS = [
    'accept',
    'accept-encoding',
    'authorization',
    'content-type',
    'origin',
]
```

### Rate Limiting

```python
# Using django-ratelimit
from django_ratelimit.decorators import ratelimit

@ratelimit(key='ip', rate='5/m', method='POST')
def login_view(request):
    """Rate limit login attempts to 5 per minute per IP."""
    pass

@ratelimit(key='user', rate='100/h')
def api_view(request):
    """Rate limit API calls to 100 per hour per user."""
    pass
```

## Session Management

### Token Lifetime

```python
# settings.py
from datetime import timedelta

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
}

# Token expiration (optional)
REST_AUTH_TOKEN_TTL = timedelta(days=7)  # 7-day tokens
```

### Concurrent Sessions

WiWebb uses a **single token per user** model:

```python
# On login, existing token is reused or recreated
token, created = Token.objects.get_or_create(user=user)

# To force new token (logout other sessions)
Token.objects.filter(user=user).delete()
token = Token.objects.create(user=user)
```

### Session Invalidation

```mermaid
graph LR
    User[User Action] -->|Logout| Delete1[Delete Token from DB]
    User -->|Password Change| Delete2[Delete Token from DB]
    User -->|Account Deactivation| Delete3[Delete Token from DB]
    Admin[Admin Action] -->|Deactivate User| Delete4[Delete Token from DB]

    Delete1 --> Invalidate[Token Invalidated]
    Delete2 --> Invalidate
    Delete3 --> Invalidate
    Delete4 --> Invalidate

    Invalidate --> NextRequest[Next API Request]
    NextRequest --> 401[401 Unauthorized]
```

## Authentication Endpoints

### Available Endpoints

| Endpoint | Method | Purpose | Auth Required |
|----------|--------|---------|---------------|
| `/auth/login/` | POST | User login | ❌ No |
| `/auth/logout/` | POST | User logout | ✅ Yes |
| `/auth/registration/` | POST | New user signup | ❌ No |
| `/auth/user/` | GET | Current user info | ✅ Yes |
| `/auth/user/` | PUT/PATCH | Update profile | ✅ Yes |
| `/auth/password/change/` | POST | Change password | ✅ Yes |
| `/auth/password/reset/` | POST | Request reset | ❌ No |
| `/auth/password/reset/confirm/` | POST | Confirm reset | ❌ No |

### Example Requests

=== "Login"
    ```bash
    curl -X POST http://localhost:8000/api/v1/auth/login/ \
      -H "Content-Type: application/json" \
      -d '{"username": "admin", "password": "password123"}'
    ```

    Response:
    ```json
    {
      "key": "9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b",
      "user": {
        "id": 1,
        "username": "admin",
        "email": "admin@example.com",
        "role": "superadmin"
      }
    }
    ```

=== "Get User"
    ```bash
    curl -X GET http://localhost:8000/api/v1/auth/user/ \
      -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
    ```

    Response:
    ```json
    {
      "id": 1,
      "username": "admin",
      "email": "admin@example.com",
      "first_name": "Admin",
      "last_name": "User",
      "role": "superadmin",
      "tenant": null,
      "is_active": true
    }
    ```

=== "Logout"
    ```bash
    curl -X POST http://localhost:8000/api/v1/auth/logout/ \
      -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
    ```

    Response:
    ```json
    {
      "detail": "Successfully logged out."
    }
    ```

## Troubleshooting

??? warning "401 Unauthorized"
    **Symptoms:** All authenticated requests return 401

    **Causes:**
    - Invalid or expired token
    - Token not included in request
    - Token format incorrect

    **Solutions:**
    ```bash
    # Check token format
    Authorization: Token 9944b09199c62bcf...  ✅ Correct
    Authorization: Bearer 9944b09199c62bcf... ❌ Wrong (not Bearer)
    Authorization: 9944b09199c62bcf...       ❌ Wrong (no prefix)
    ```

??? warning "403 Forbidden"
    **Symptoms:** Request authenticated but denied

    **Causes:**
    - Insufficient role permissions
    - Accessing resource outside tenant scope
    - Object-level permission denied

    **Solutions:**
    - Check user role in `/auth/user/`
    - Verify tenant association
    - Review permission matrix above

## Next Steps

- **[Multi-Tenancy Architecture](multi-tenancy.md)** - Tenant isolation details
- **[API Reference](../api-reference/authentication.md)** - Full auth API documentation
- **[User Guide](../user-guide/organizations/roles.md)** - Role management guide

---

!!! question "Security Questions?"
    For security-related inquiries, contact security@thinesoft.com
