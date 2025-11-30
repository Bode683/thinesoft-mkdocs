# Authentication API

Complete API reference for WiWebb authentication endpoints.

## Base URL

```
http://localhost:8000/api/v1/auth/
```

!!! info "Production URL"
    Replace `localhost:8000` with your production domain.

## Authentication

All endpoints except login and registration require authentication using a token:

```
Authorization: Token {your-auth-token}
```

## Endpoints Overview

| Endpoint | Method | Auth Required | Description |
|----------|--------|---------------|-------------|
| `/login/` | POST | ❌ | User login |
| `/logout/` | POST | ✅ | User logout |
| `/registration/` | POST | ❌ | User registration |
| `/user/` | GET | ✅ | Get current user |
| `/user/` | PUT/PATCH | ✅ | Update profile |
| `/password/change/` | POST | ✅ | Change password |
| `/password/reset/` | POST | ❌ | Request password reset |
| `/password/reset/confirm/` | POST | ❌ | Confirm password reset |

---

## Login

Authenticate a user and receive an authentication token.

### Endpoint

```
POST /api/v1/auth/login/
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| username | string | ✅ | Username or email |
| password | string | ✅ | User password |

### Example Requests

=== "cURL"
    ```bash
    curl -X POST http://localhost:8000/api/v1/auth/login/ \
      -H "Content-Type: application/json" \
      -d '{
        "username": "admin",
        "password": "password123"
      }'
    ```

=== "Python (requests)"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/auth/login/"
    payload = {
        "username": "admin",
        "password": "password123"
    }

    response = requests.post(url, json=payload)
    data = response.json()

    # Store the token
    token = data['key']
    print(f"Token: {token}")
    ```

=== "JavaScript (fetch)"
    ```javascript
    const response = await fetch('http://localhost:8000/api/v1/auth/login/', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        username: 'admin',
        password: 'password123'
      })
    });

    const data = await response.json();
    // Store the token
    localStorage.setItem('auth_token', data.key);
    console.log('Token:', data.key);
    ```

=== "TypeScript"
    ```typescript
    interface LoginResponse {
      key: string;
      user: {
        id: number;
        username: string;
        email: string;
        role: string;
      };
    }

    async function login(username: string, password: string): Promise<LoginResponse> {
      const response = await fetch('http://localhost:8000/api/v1/auth/login/', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ username, password })
      });

      if (!response.ok) {
        throw new Error('Login failed');
      }

      return response.json();
    }

    // Usage
    const { key, user } = await login('admin', 'password123');
    localStorage.setItem('auth_token', key);
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "key": "9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
}
```

### Error Responses

**Code:** `400 Bad Request`

```json
{
  "non_field_errors": [
    "Unable to log in with provided credentials."
  ]
}
```

**Code:** `400 Bad Request` (Missing fields)

```json
{
  "username": ["This field is required."],
  "password": ["This field is required."]
}
```

---

## Logout

Invalidate the current authentication token.

### Endpoint

```
POST /api/v1/auth/logout/
```

### Headers

```
Authorization: Token {your-auth-token}
```

### Example Requests

=== "cURL"
    ```bash
    curl -X POST http://localhost:8000/api/v1/auth/logout/ \
      -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/auth/logout/"
    headers = {
        "Authorization": "Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
    }

    response = requests.post(url, headers=headers)
    print(response.json())
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    const response = await fetch('http://localhost:8000/api/v1/auth/logout/', {
      method: 'POST',
      headers: {
        'Authorization': `Token ${token}`
      }
    });

    if (response.ok) {
      // Remove token from storage
      localStorage.removeItem('auth_token');
      console.log('Logged out successfully');
    }
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "detail": "Successfully logged out."
}
```

### Error Responses

**Code:** `401 Unauthorized`

```json
{
  "detail": "Invalid token."
}
```

---

## Registration

Register a new user account.

### Endpoint

```
POST /api/v1/auth/registration/
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| username | string | ✅ | Desired username (unique) |
| email | string | ✅ | Email address (unique) |
| password1 | string | ✅ | Password |
| password2 | string | ✅ | Password confirmation |
| first_name | string | ❌ | First name |
| last_name | string | ❌ | Last name |

### Example Requests

=== "cURL"
    ```bash
    curl -X POST http://localhost:8000/api/v1/auth/registration/ \
      -H "Content-Type: application/json" \
      -d '{
        "username": "newuser",
        "email": "newuser@example.com",
        "password1": "SecurePassword123!",
        "password2": "SecurePassword123!",
        "first_name": "John",
        "last_name": "Doe"
      }'
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/auth/registration/"
    payload = {
        "username": "newuser",
        "email": "newuser@example.com",
        "password1": "SecurePassword123!",
        "password2": "SecurePassword123!",
        "first_name": "John",
        "last_name": "Doe"
    }

    response = requests.post(url, json=payload)
    data = response.json()

    # User is registered and logged in
    token = data['key']
    print(f"Registration successful! Token: {token}")
    ```

=== "JavaScript"
    ```javascript
    const response = await fetch('http://localhost:8000/api/v1/auth/registration/', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        username: 'newuser',
        email: 'newuser@example.com',
        password1: 'SecurePassword123!',
        password2: 'SecurePassword123!',
        first_name: 'John',
        last_name: 'Doe'
      })
    });

    const data = await response.json();
    // User is automatically logged in
    localStorage.setItem('auth_token', data.key);
    ```

### Success Response

**Code:** `201 Created`

```json
{
  "key": "abc123def456789...",
  "user": {
    "id": 42,
    "username": "newuser",
    "email": "newuser@example.com",
    "first_name": "John",
    "last_name": "Doe"
  }
}
```

### Error Responses

**Code:** `400 Bad Request` (Passwords don't match)

```json
{
  "password2": ["The two password fields didn't match."]
}
```

**Code:** `400 Bad Request` (Username taken)

```json
{
  "username": ["A user with that username already exists."]
}
```

**Code:** `400 Bad Request` (Weak password)

```json
{
  "password1": [
    "This password is too short. It must contain at least 8 characters.",
    "This password is too common."
  ]
}
```

---

## Get Current User

Retrieve the authenticated user's profile.

### Endpoint

```
GET /api/v1/auth/user/
```

### Headers

```
Authorization: Token {your-auth-token}
```

### Example Requests

=== "cURL"
    ```bash
    curl -X GET http://localhost:8000/api/v1/auth/user/ \
      -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/auth/user/"
    headers = {
        "Authorization": "Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
    }

    response = requests.get(url, headers=headers)
    user = response.json()
    print(f"Logged in as: {user['username']}")
    print(f"Role: {user['role']}")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    const response = await fetch('http://localhost:8000/api/v1/auth/user/', {
      headers: {
        'Authorization': `Token ${token}`
      }
    });

    const user = await response.json();
    console.log('Current user:', user);
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "id": 1,
  "username": "admin",
  "email": "admin@example.com",
  "first_name": "Admin",
  "last_name": "User",
  "role": "superadmin",
  "tenant": null,
  "tenant_name": null,
  "is_active": true,
  "phone_number": "+1234567890",
  "bio": "System administrator",
  "url": "https://example.com",
  "company": "ACME Corp",
  "location": "San Francisco, CA",
  "date_joined": "2025-01-01T00:00:00Z",
  "last_login": "2025-11-30T10:30:00Z"
}
```

### Error Responses

**Code:** `401 Unauthorized`

```json
{
  "detail": "Invalid token."
}
```

---

## Update Profile

Update the authenticated user's profile.

### Endpoint

```
PUT /api/v1/auth/user/     # Full update
PATCH /api/v1/auth/user/   # Partial update
```

### Headers

```
Authorization: Token {your-auth-token}
Content-Type: application/json
```

### Request Body

All fields are optional for PATCH requests:

| Field | Type | Description |
|-------|------|-------------|
| first_name | string | First name |
| last_name | string | Last name |
| email | string | Email address |
| phone_number | string | Phone number |
| bio | string | Biography |
| url | string | Website URL |
| company | string | Company name |
| location | string | Location |

!!! warning "Read-Only Fields"
    Cannot update: `id`, `username`, `role`, `tenant`, `date_joined`, `last_login`

### Example Requests

=== "cURL"
    ```bash
    curl -X PATCH http://localhost:8000/api/v1/auth/user/ \
      -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b" \
      -H "Content-Type: application/json" \
      -d '{
        "first_name": "John",
        "last_name": "Smith",
        "phone_number": "+1234567890"
      }'
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/auth/user/"
    headers = {
        "Authorization": "Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b",
        "Content-Type": "application/json"
    }
    payload = {
        "first_name": "John",
        "last_name": "Smith",
        "phone_number": "+1234567890"
    }

    response = requests.patch(url, headers=headers, json=payload)
    user = response.json()
    print(f"Profile updated: {user['first_name']} {user['last_name']}")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    const response = await fetch('http://localhost:8000/api/v1/auth/user/', {
      method: 'PATCH',
      headers: {
        'Authorization': `Token ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        first_name: 'John',
        last_name: 'Smith',
        phone_number: '+1234567890'
      })
    });

    const user = await response.json();
    console.log('Profile updated:', user);
    ```

### Success Response

**Code:** `200 OK`

Returns the updated user object (same format as GET `/auth/user/`).

---

## Change Password

Change the authenticated user's password.

### Endpoint

```
POST /api/v1/auth/password/change/
```

### Headers

```
Authorization: Token {your-auth-token}
Content-Type: application/json
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| old_password | string | ✅ | Current password |
| new_password1 | string | ✅ | New password |
| new_password2 | string | ✅ | New password confirmation |

### Example Requests

=== "cURL"
    ```bash
    curl -X POST http://localhost:8000/api/v1/auth/password/change/ \
      -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b" \
      -H "Content-Type: application/json" \
      -d '{
        "old_password": "OldPassword123",
        "new_password1": "NewSecurePassword456!",
        "new_password2": "NewSecurePassword456!"
      }'
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/auth/password/change/"
    headers = {
        "Authorization": "Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b",
        "Content-Type": "application/json"
    }
    payload = {
        "old_password": "OldPassword123",
        "new_password1": "NewSecurePassword456!",
        "new_password2": "NewSecurePassword456!"
    }

    response = requests.post(url, headers=headers, json=payload)
    print(response.json())
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    const response = await fetch('http://localhost:8000/api/v1/auth/password/change/', {
      method: 'POST',
      headers: {
        'Authorization': `Token ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        old_password: 'OldPassword123',
        new_password1: 'NewSecurePassword456!',
        new_password2: 'NewSecurePassword456!'
      })
    });

    const data = await response.json();
    console.log('Password changed:', data);
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "detail": "New password has been saved."
}
```

### Error Responses

**Code:** `400 Bad Request` (Wrong old password)

```json
{
  "old_password": ["Wrong password."]
}
```

**Code:** `400 Bad Request` (Passwords don't match)

```json
{
  "new_password2": ["The two password fields didn't match."]
}
```

---

## Request Password Reset

Request a password reset email.

### Endpoint

```
POST /api/v1/auth/password/reset/
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| email | string | ✅ | User's email address |

### Example Requests

=== "cURL"
    ```bash
    curl -X POST http://localhost:8000/api/v1/auth/password/reset/ \
      -H "Content-Type: application/json" \
      -d '{
        "email": "user@example.com"
      }'
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/auth/password/reset/"
    payload = {"email": "user@example.com"}

    response = requests.post(url, json=payload)
    print(response.json())
    ```

=== "JavaScript"
    ```javascript
    const response = await fetch('http://localhost:8000/api/v1/auth/password/reset/', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        email: 'user@example.com'
      })
    });

    const data = await response.json();
    console.log(data);
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "detail": "Password reset e-mail has been sent."
}
```

!!! info "Email Configuration Required"
    Email must be configured in Django settings for this endpoint to work.
    See [Configuration Guide](../getting-started/configuration.md#email-configuration).

---

## SDK Examples

### Python Client

```python title="wiwebb_client.py"
import requests

class WiwebbClient:
    """Python client for WiWebb API."""

    def __init__(self, base_url="http://localhost:8000", token=None):
        self.base_url = base_url
        self.token = token

    def login(self, username, password):
        """Login and store token."""
        response = requests.post(
            f"{self.base_url}/api/v1/auth/login/",
            json={"username": username, "password": password}
        )
        response.raise_for_status()
        data = response.json()
        self.token = data['key']
        return data

    def get_headers(self):
        """Get headers with auth token."""
        if not self.token:
            raise ValueError("Not authenticated. Call login() first.")
        return {"Authorization": f"Token {self.token}"}

    def get_current_user(self):
        """Get current user profile."""
        response = requests.get(
            f"{self.base_url}/api/v1/auth/user/",
            headers=self.get_headers()
        )
        response.raise_for_status()
        return response.json()

    def update_profile(self, **kwargs):
        """Update user profile."""
        response = requests.patch(
            f"{self.base_url}/api/v1/auth/user/",
            headers=self.get_headers(),
            json=kwargs
        )
        response.raise_for_status()
        return response.json()

    def logout(self):
        """Logout and clear token."""
        response = requests.post(
            f"{self.base_url}/api/v1/auth/logout/",
            headers=self.get_headers()
        )
        response.raise_for_status()
        self.token = None
        return response.json()

# Usage
client = WiwebbClient()
client.login("admin", "password123")
user = client.get_current_user()
print(f"Logged in as: {user['username']}")
```

### TypeScript Client

```typescript title="wiwebbClient.ts"
interface LoginResponse {
  key: string;
}

interface User {
  id: number;
  username: string;
  email: string;
  role: string;
  [key: string]: any;
}

class WiwebbClient {
  private baseUrl: string;
  private token: string | null = null;

  constructor(baseUrl: string = 'http://localhost:8000') {
    this.baseUrl = baseUrl;
  }

  async login(username: string, password: string): Promise<LoginResponse> {
    const response = await fetch(`${this.baseUrl}/api/v1/auth/login/`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password })
    });

    if (!response.ok) {
      throw new Error('Login failed');
    }

    const data: LoginResponse = await response.json();
    this.token = data.key;
    return data;
  }

  private getHeaders(): HeadersInit {
    if (!this.token) {
      throw new Error('Not authenticated');
    }
    return {
      'Authorization': `Token ${this.token}`,
      'Content-Type': 'application/json'
    };
  }

  async getCurrentUser(): Promise<User> {
    const response = await fetch(`${this.baseUrl}/api/v1/auth/user/`, {
      headers: this.getHeaders()
    });

    if (!response.ok) {
      throw new Error('Failed to get user');
    }

    return response.json();
  }

  async updateProfile(updates: Partial<User>): Promise<User> {
    const response = await fetch(`${this.baseUrl}/api/v1/auth/user/`, {
      method: 'PATCH',
      headers: this.getHeaders(),
      body: JSON.stringify(updates)
    });

    if (!response.ok) {
      throw new Error('Failed to update profile');
    }

    return response.json();
  }

  async logout(): Promise<void> {
    await fetch(`${this.baseUrl}/api/v1/auth/logout/`, {
      method: 'POST',
      headers: this.getHeaders()
    });

    this.token = null;
  }
}

// Usage
const client = new WiwebbClient();
await client.login('admin', 'password123');
const user = await client.getCurrentUser();
console.log(`Logged in as: ${user.username}`);
```

## Next Steps

- **[Tenants API](tenants.md)** - Tenant management endpoints
- **[Users API](users.md)** - User management endpoints
- **[Subscriptions API](subscriptions.md)** - Billing and subscriptions

---

!!! question "Need Help?"
    For API questions, check the [FAQ](../troubleshooting/faq.md) or contact dev@thinesoft.com
