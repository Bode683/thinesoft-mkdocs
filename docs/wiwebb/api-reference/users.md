# Users API

Complete API reference for managing users in WiWebb.

## Base URL

```
http://localhost:8000/api/v1/users/
```

## Authentication

All endpoints require authentication:

```
Authorization: Token {your-auth-token}
```

## Permissions

| Role | List All | List Tenant | Create Any | Create Tenant | Update Any | Update Own | Delete | Assign Role |
|------|:--------:|:-----------:|:----------:|:-------------:|:----------:|:----------:|:------:|:-----------:|
| SuperAdmin | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Any role |
| Admin | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Except SuperAdmin |
| Tenant Owner | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ | Only Subscriber |
| Subscriber | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | None |

## Endpoints Overview

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/users/` | GET | List users |
| `/users/` | POST | Create user |
| `/users/{id}/` | GET | Retrieve user |
| `/users/{id}/` | PUT/PATCH | Update user |
| `/users/{id}/` | DELETE | Delete user |
| `/users/{id}/assign-role/` | POST | Assign role to user |
| `/users/{id}/set-password/` | POST | Set user password |
| `/users/{id}/activate/` | POST | Activate/deactivate user |

---

## List Users

Retrieve a list of users based on permissions.

### Endpoint

```
GET /api/v1/users/
```

### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| page | integer | Page number |
| page_size | integer | Items per page (default: 10) |
| search | string | Search in username, email, name |
| role | string | Filter by role |
| tenant | integer | Filter by tenant ID |
| is_active | boolean | Filter by active status |
| ordering | string | Order by field (e.g., `username`, `-date_joined`) |

### Example Requests

=== "cURL"
    ```bash
    # List all users (admin)
    curl -X GET http://localhost:8000/api/v1/users/ \
      -H "Authorization: Token {your-token}"

    # Filter by tenant and role
    curl -X GET "http://localhost:8000/api/v1/users/?tenant=1&role=subscriber" \
      -H "Authorization: Token {your-token}"

    # Search with pagination
    curl -X GET "http://localhost:8000/api/v1/users/?search=john&page_size=20" \
      -H "Authorization: Token {your-token}"
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/users/"
    headers = {"Authorization": "Token {your-token}"}

    # List all users
    response = requests.get(url, headers=headers)
    users = response.json()

    # Filter by tenant
    params = {"tenant": 1, "is_active": True}
    response = requests.get(url, headers=headers, params=params)

    for user in response.json()['results']:
        print(f"{user['username']} - {user['email']} ({user['role']})")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    // List users with filters
    const params = new URLSearchParams({
      tenant: '1',
      role: 'subscriber',
      is_active: 'true',
      ordering: 'username'
    });

    const response = await fetch(
      `http://localhost:8000/api/v1/users/?${params}`,
      {
        headers: { 'Authorization': `Token ${token}` }
      }
    );

    const data = await response.json();
    console.log(`Found ${data.count} users`);
    data.results.forEach(user => {
      console.log(`${user.username} (${user.role})`);
    });
    ```

=== "TypeScript"
    ```typescript
    interface User {
      id: number;
      username: string;
      email: string;
      first_name: string;
      last_name: string;
      role: 'superadmin' | 'admin' | 'tenant_owner' | 'subscriber';
      tenant: number | null;
      tenant_name: string | null;
      is_active: boolean;
      phone_number: string;
      bio: string;
      company: string;
      location: string;
      date_joined: string;
      last_login: string;
    }

    interface UserListParams {
      page?: number;
      page_size?: number;
      search?: string;
      role?: string;
      tenant?: number;
      is_active?: boolean;
      ordering?: string;
    }

    async function listUsers(params?: UserListParams) {
      const token = localStorage.getItem('auth_token');
      const queryParams = new URLSearchParams(
        params as Record<string, string>
      );

      const response = await fetch(
        `http://localhost:8000/api/v1/users/?${queryParams}`,
        {
          headers: { 'Authorization': `Token ${token}` }
        }
      );

      if (!response.ok) {
        throw new Error('Failed to fetch users');
      }

      return response.json();
    }

    // Usage
    const { results, count } = await listUsers({
      tenant: 1,
      is_active: true,
      ordering: 'username'
    });
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "count": 42,
  "next": "http://localhost:8000/api/v1/users/?page=2",
  "previous": null,
  "results": [
    {
      "id": 5,
      "username": "john.doe",
      "email": "john.doe@example.com",
      "first_name": "John",
      "last_name": "Doe",
      "role": "subscriber",
      "tenant": 1,
      "tenant_name": "ACME Corporation",
      "is_active": true,
      "phone_number": "+1234567890",
      "bio": "Software engineer",
      "url": "https://johndoe.dev",
      "company": "ACME Corp",
      "location": "San Francisco, CA",
      "date_joined": "2025-03-15T10:00:00Z",
      "last_login": "2025-11-30T09:30:00Z"
    }
  ]
}
```

---

## Create User

Create a new user account.

### Endpoint

```
POST /api/v1/users/
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| username | string | ✅ | Unique username |
| email | string | ✅ | Unique email address |
| password | string | ✅ | User password |
| first_name | string | ❌ | First name |
| last_name | string | ❌ | Last name |
| role | string | ✅ | User role (enum) |
| tenant | integer | ❌ | Tenant ID (required for non-admin roles) |
| phone_number | string | ❌ | Phone number |
| bio | string | ❌ | Biography |
| company | string | ❌ | Company name |
| location | string | ❌ | Location |
| is_active | boolean | ❌ | Active status (default: true) |

**Role Values:**
- `superadmin` - SuperAdmin (only SuperAdmin can assign)
- `admin` - Admin (SuperAdmin/Admin can assign)
- `tenant_owner` - Tenant Owner (SuperAdmin/Admin can assign)
- `subscriber` - Subscriber (all can assign)

### Example Requests

=== "cURL"
    ```bash
    curl -X POST http://localhost:8000/api/v1/users/ \
      -H "Authorization: Token {your-token}" \
      -H "Content-Type: application/json" \
      -d '{
        "username": "jane.smith",
        "email": "jane.smith@example.com",
        "password": "SecurePassword123!",
        "first_name": "Jane",
        "last_name": "Smith",
        "role": "subscriber",
        "tenant": 1,
        "phone_number": "+1987654321",
        "company": "ACME Corp"
      }'
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/users/"
    headers = {
        "Authorization": "Token {your-token}",
        "Content-Type": "application/json"
    }

    user_data = {
        "username": "jane.smith",
        "email": "jane.smith@example.com",
        "password": "SecurePassword123!",
        "first_name": "Jane",
        "last_name": "Smith",
        "role": "subscriber",
        "tenant": 1,
        "phone_number": "+1987654321",
        "company": "ACME Corp"
    }

    response = requests.post(url, headers=headers, json=user_data)

    if response.status_code == 201:
        user = response.json()
        print(f"Created user: {user['username']} (ID: {user['id']})")
    else:
        print(f"Error: {response.json()}")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    const userData = {
      username: 'jane.smith',
      email: 'jane.smith@example.com',
      password: 'SecurePassword123!',
      first_name: 'Jane',
      last_name: 'Smith',
      role: 'subscriber',
      tenant: 1,
      phone_number: '+1987654321',
      company: 'ACME Corp'
    };

    const response = await fetch('http://localhost:8000/api/v1/users/', {
      method: 'POST',
      headers: {
        'Authorization': `Token ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(userData)
    });

    if (response.ok) {
      const user = await response.json();
      console.log('User created:', user);
    } else {
      const error = await response.json();
      console.error('Error:', error);
    }
    ```

### Success Response

**Code:** `201 Created`

```json
{
  "id": 15,
  "username": "jane.smith",
  "email": "jane.smith@example.com",
  "first_name": "Jane",
  "last_name": "Smith",
  "role": "subscriber",
  "tenant": 1,
  "tenant_name": "ACME Corporation",
  "is_active": true,
  "phone_number": "+1987654321",
  "bio": "",
  "url": "",
  "company": "ACME Corp",
  "location": "",
  "date_joined": "2025-11-30T16:00:00Z",
  "last_login": null
}
```

### Error Responses

**Code:** `400 Bad Request` (Username exists)

```json
{
  "username": ["A user with that username already exists."]
}
```

**Code:** `400 Bad Request` (Email exists)

```json
{
  "email": ["user with this email already exists."]
}
```

**Code:** `403 Forbidden` (Insufficient permissions)

```json
{
  "detail": "Cannot assign SuperAdmin role."
}
```

---

## Retrieve User

Get details of a specific user.

### Endpoint

```
GET /api/v1/users/{id}/
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| id | integer | User ID |

### Example Requests

=== "cURL"
    ```bash
    curl -X GET http://localhost:8000/api/v1/users/5/ \
      -H "Authorization: Token {your-token}"
    ```

=== "Python"
    ```python
    import requests

    user_id = 5
    url = f"http://localhost:8000/api/v1/users/{user_id}/"
    headers = {"Authorization": "Token {your-token}"}

    response = requests.get(url, headers=headers)
    user = response.json()

    print(f"Username: {user['username']}")
    print(f"Email: {user['email']}")
    print(f"Role: {user['role']}")
    print(f"Tenant: {user['tenant_name']}")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');
    const userId = 5;

    const response = await fetch(
      `http://localhost:8000/api/v1/users/${userId}/`,
      {
        headers: { 'Authorization': `Token ${token}` }
      }
    );

    const user = await response.json();
    console.log('User details:', user);
    ```

### Success Response

**Code:** `200 OK`

Returns user object (same format as in list response).

---

## Update User

Update an existing user.

### Endpoint

```
PUT /api/v1/users/{id}/     # Full update
PATCH /api/v1/users/{id}/   # Partial update
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| id | integer | User ID |

### Request Body

All fields optional for PATCH:

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
| is_active | boolean | Active status |

!!! warning "Read-Only Fields"
    Cannot update directly: `username`, `role`, `tenant`, `date_joined`, `last_login`

    Use dedicated endpoints for:
    - Role changes: `/users/{id}/assign-role/`
    - Password changes: `/users/{id}/set-password/`

### Example Requests

=== "cURL"
    ```bash
    curl -X PATCH http://localhost:8000/api/v1/users/5/ \
      -H "Authorization: Token {your-token}" \
      -H "Content-Type: application/json" \
      -d '{
        "phone_number": "+1111111111",
        "location": "New York, NY"
      }'
    ```

=== "Python"
    ```python
    import requests

    user_id = 5
    url = f"http://localhost:8000/api/v1/users/{user_id}/"
    headers = {
        "Authorization": "Token {your-token}",
        "Content-Type": "application/json"
    }

    updates = {
        "phone_number": "+1111111111",
        "location": "New York, NY"
    }

    response = requests.patch(url, headers=headers, json=updates)
    updated_user = response.json()
    print(f"Updated: {updated_user['username']}")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');
    const userId = 5;

    const response = await fetch(
      `http://localhost:8000/api/v1/users/${userId}/`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Token ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          phone_number: '+1111111111',
          location: 'New York, NY'
        })
      }
    );

    const user = await response.json();
    console.log('Updated user:', user);
    ```

### Success Response

**Code:** `200 OK`

Returns the updated user object.

---

## Delete User

Delete a user account.

!!! danger "Permanent Action"
    This operation cannot be undone. All user data will be deleted.

### Endpoint

```
DELETE /api/v1/users/{id}/
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| id | integer | User ID |

### Example Requests

=== "cURL"
    ```bash
    curl -X DELETE http://localhost:8000/api/v1/users/15/ \
      -H "Authorization: Token {your-token}"
    ```

=== "Python"
    ```python
    import requests

    user_id = 15
    url = f"http://localhost:8000/api/v1/users/{user_id}/"
    headers = {"Authorization": "Token {your-token}"}

    response = requests.delete(url, headers=headers)

    if response.status_code == 204:
        print(f"User {user_id} deleted successfully")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');
    const userId = 15;

    const response = await fetch(
      `http://localhost:8000/api/v1/users/${userId}/`,
      {
        method: 'DELETE',
        headers: { 'Authorization': `Token ${token}` }
      }
    );

    if (response.ok) {
      console.log('User deleted successfully');
    }
    ```

### Success Response

**Code:** `204 No Content`

(No response body)

### Error Response

**Code:** `403 Forbidden`

```json
{
  "detail": "You do not have permission to perform this action."
}
```

---

## Assign Role

Change a user's role.

!!! info "Role Assignment Rules"
    - SuperAdmin can assign any role
    - Admin can assign any role except SuperAdmin
    - Tenant Owner can only assign Subscriber role
    - Cannot cross-tenant role assignment

### Endpoint

```
POST /api/v1/users/{id}/assign-role/
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| id | integer | User ID |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| role | string | ✅ | New role (superadmin/admin/tenant_owner/subscriber) |

### Example Requests

=== "cURL"
    ```bash
    curl -X POST http://localhost:8000/api/v1/users/5/assign-role/ \
      -H "Authorization: Token {your-token}" \
      -H "Content-Type: application/json" \
      -d '{
        "role": "tenant_owner"
      }'
    ```

=== "Python"
    ```python
    import requests

    user_id = 5
    url = f"http://localhost:8000/api/v1/users/{user_id}/assign-role/"
    headers = {
        "Authorization": "Token {your-token}",
        "Content-Type": "application/json"
    }

    response = requests.post(
        url,
        headers=headers,
        json={"role": "tenant_owner"}
    )

    result = response.json()
    print(result['message'])
    print(f"New role: {result['user']['role']}")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');
    const userId = 5;

    const response = await fetch(
      `http://localhost:8000/api/v1/users/${userId}/assign-role/`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Token ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          role: 'tenant_owner'
        })
      }
    );

    const result = await response.json();
    console.log(result.message);
    console.log('Updated user:', result.user);
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "message": "Role updated successfully",
  "user": {
    "id": 5,
    "username": "john.doe",
    "email": "john.doe@example.com",
    "role": "tenant_owner",
    "tenant": 1,
    "tenant_name": "ACME Corporation"
  }
}
```

### Error Responses

**Code:** `400 Bad Request` (Invalid role)

```json
{
  "error": "Invalid role"
}
```

**Code:** `403 Forbidden` (Cannot assign role)

```json
{
  "error": "Cannot assign SuperAdmin role"
}
```

**Code:** `403 Forbidden` (Cross-tenant)

```json
{
  "error": "Cannot modify users from other tenants"
}
```

---

## Set Password

Set a user's password (admin operation).

### Endpoint

```
POST /api/v1/users/{id}/set-password/
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| id | integer | User ID |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| password | string | ✅ | New password |

### Example Requests

=== "cURL"
    ```bash
    curl -X POST http://localhost:8000/api/v1/users/5/set-password/ \
      -H "Authorization: Token {your-token}" \
      -H "Content-Type: application/json" \
      -d '{
        "password": "NewSecurePassword456!"
      }'
    ```

=== "Python"
    ```python
    import requests

    user_id = 5
    url = f"http://localhost:8000/api/v1/users/{user_id}/set-password/"
    headers = {
        "Authorization": "Token {your-token}",
        "Content-Type": "application/json"
    }

    response = requests.post(
        url,
        headers=headers,
        json={"password": "NewSecurePassword456!"}
    )

    print(response.json()['message'])
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');
    const userId = 5;

    const response = await fetch(
      `http://localhost:8000/api/v1/users/${userId}/set-password/`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Token ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          password: 'NewSecurePassword456!'
        })
      }
    );

    const result = await response.json();
    console.log(result.message);
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "message": "Password updated successfully"
}
```

### Error Response

**Code:** `400 Bad Request` (Weak password)

```json
{
  "password": [
    "This password is too short. It must contain at least 8 characters."
  ]
}
```

---

## Activate/Deactivate User

Change a user's active status.

### Endpoint

```
POST /api/v1/users/{id}/activate/
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| id | integer | User ID |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| is_active | boolean | ✅ | Active status (true/false) |

### Example Requests

=== "cURL"
    ```bash
    # Deactivate user
    curl -X POST http://localhost:8000/api/v1/users/5/activate/ \
      -H "Authorization: Token {your-token}" \
      -H "Content-Type: application/json" \
      -d '{
        "is_active": false
      }'
    ```

=== "Python"
    ```python
    import requests

    def toggle_user_status(user_id, active):
        """Activate or deactivate a user."""
        url = f"http://localhost:8000/api/v1/users/{user_id}/activate/"
        headers = {
            "Authorization": "Token {your-token}",
            "Content-Type": "application/json"
        }

        response = requests.post(
            url,
            headers=headers,
            json={"is_active": active}
        )

        result = response.json()
        status = "activated" if active else "deactivated"
        print(f"User {status}: {result['user']['username']}")
        return result

    # Deactivate user
    toggle_user_status(5, False)

    # Reactivate user
    toggle_user_status(5, True)
    ```

=== "JavaScript"
    ```javascript
    async function toggleUserStatus(userId, isActive) {
      const token = localStorage.getItem('auth_token');

      const response = await fetch(
        `http://localhost:8000/api/v1/users/${userId}/activate/`,
        {
          method: 'POST',
          headers: {
            'Authorization': `Token ${token}`,
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ is_active: isActive })
        }
      );

      const result = await response.json();
      console.log(result.message);
      return result;
    }

    // Deactivate
    await toggleUserStatus(5, false);

    // Activate
    await toggleUserStatus(5, true);
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "message": "User deactivated successfully",
  "user": {
    "id": 5,
    "username": "john.doe",
    "is_active": false
  }
}
```

---

## Bulk Operations

### Create Multiple Users

=== "Python"
    ```python
    import requests

    def bulk_create_users(users_data):
        """Create multiple users in bulk."""
        url = "http://localhost:8000/api/v1/users/"
        headers = {
            "Authorization": "Token {your-token}",
            "Content-Type": "application/json"
        }

        created = []
        failed = []

        for user_data in users_data:
            response = requests.post(url, headers=headers, json=user_data)

            if response.status_code == 201:
                user = response.json()
                created.append(user)
                print(f"✓ Created: {user['username']}")
            else:
                failed.append({
                    'data': user_data,
                    'error': response.json()
                })
                print(f"✗ Failed: {user_data['username']}")

        print(f"\nCreated: {len(created)}, Failed: {len(failed)}")
        return {"created": created, "failed": failed}

    # Usage
    users = [
        {
            "username": "user1",
            "email": "user1@example.com",
            "password": "Password123!",
            "role": "subscriber",
            "tenant": 1
        },
        {
            "username": "user2",
            "email": "user2@example.com",
            "password": "Password123!",
            "role": "subscriber",
            "tenant": 1
        }
    ]

    result = bulk_create_users(users)
    ```

### Import Users from CSV

=== "Python"
    ```python
    import csv
    import requests

    def import_users_from_csv(csv_file, tenant_id):
        """Import users from CSV file."""
        url = "http://localhost:8000/api/v1/users/"
        headers = {
            "Authorization": "Token {your-token}",
            "Content-Type": "application/json"
        }

        with open(csv_file, 'r') as file:
            reader = csv.DictReader(file)

            for row in reader:
                user_data = {
                    "username": row['username'],
                    "email": row['email'],
                    "password": row.get('password', 'ChangeMe123!'),
                    "first_name": row.get('first_name', ''),
                    "last_name": row.get('last_name', ''),
                    "role": row.get('role', 'subscriber'),
                    "tenant": tenant_id
                }

                response = requests.post(url, headers=headers, json=user_data)

                if response.status_code == 201:
                    print(f"✓ Imported: {user_data['username']}")
                else:
                    print(f"✗ Failed: {user_data['username']} - {response.json()}")

    # Usage
    import_users_from_csv('users.csv', tenant_id=1)
    ```

## Common Use Cases

### Use Case 1: Create Tenant Owner

```python
def create_tenant_with_owner(tenant_name, owner_email):
    """Create a tenant and its owner in one workflow."""
    base_url = "http://localhost:8000/api/v1"
    headers = {
        "Authorization": "Token {your-admin-token}",
        "Content-Type": "application/json"
    }

    # Step 1: Create tenant
    tenant_data = {
        "name": tenant_name,
        "slug": tenant_name.lower().replace(" ", "-"),
        "email": owner_email
    }

    tenant_response = requests.post(
        f"{base_url}/tenants/",
        headers=headers,
        json=tenant_data
    )
    tenant = tenant_response.json()
    print(f"✓ Created tenant: {tenant['name']}")

    # Step 2: Create owner
    owner_username = owner_email.split('@')[0]
    user_data = {
        "username": owner_username,
        "email": owner_email,
        "password": "TemporaryPass123!",  # Should be changed
        "role": "tenant_owner",
        "tenant": tenant['id']
    }

    user_response = requests.post(
        f"{base_url}/users/",
        headers=headers,
        json=user_data
    )
    user = user_response.json()
    print(f"✓ Created owner: {user['username']}")

    return {"tenant": tenant, "owner": user}

# Usage
result = create_tenant_with_owner("NewCo Inc", "admin@newco.com")
```

### Use Case 2: Offboard User

```python
def offboard_user(user_id):
    """Safely offboard a user (deactivate, don't delete)."""
    base_url = "http://localhost:8000/api/v1"
    headers = {
        "Authorization": "Token {your-token}",
        "Content-Type": "application/json"
    }

    # Deactivate user
    requests.post(
        f"{base_url}/users/{user_id}/activate/",
        headers=headers,
        json={"is_active": False}
    )
    print(f"✓ User {user_id} deactivated")

    # Optionally: Revoke sessions (logout)
    # This would require additional endpoint

    return True

# Usage
offboard_user(15)
```

## Next Steps

- **[Authentication API](authentication.md)** - Auth endpoints
- **[Tenants API](tenants.md)** - Tenant management
- **[Subscriptions API](subscriptions.md)** - Billing and subscriptions
- **[User Guide](../user-guide/organizations/users.md)** - Managing users in the UI

---

!!! question "Need Help?"
    For API questions, check the [FAQ](../troubleshooting/faq.md) or contact dev@thinesoft.com
