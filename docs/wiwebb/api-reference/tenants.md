# Tenants API

Complete API reference for managing tenants (organizations) in WiWebb.

## Base URL

```
http://localhost:8000/api/v1/tenants/
```

## Authentication

All endpoints require authentication:

```
Authorization: Token {your-auth-token}
```

## Permissions

| Role | List All | List Own | Create | Update Any | Update Own | Delete |
|------|:--------:|:--------:|:------:|:----------:|:----------:|:------:|
| SuperAdmin | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Admin | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Tenant Owner | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ |
| Subscriber | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

## Endpoints Overview

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/tenants/` | GET | List tenants |
| `/tenants/` | POST | Create tenant |
| `/tenants/{id}/` | GET | Retrieve tenant |
| `/tenants/{id}/` | PUT/PATCH | Update tenant |
| `/tenants/{id}/` | DELETE | Delete tenant |
| `/tenants/me/` | GET | Get current user's tenant |

---

## List Tenants

Retrieve a list of tenants based on user permissions.

### Endpoint

```
GET /api/v1/tenants/
```

### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| page | integer | Page number for pagination |
| page_size | integer | Number of items per page (default: 10) |
| search | string | Search in name, slug, email |
| is_active | boolean | Filter by active status |
| ordering | string | Order by field (e.g., `name`, `-created_at`) |

### Example Requests

=== "cURL"
    ```bash
    # List all tenants
    curl -X GET http://localhost:8000/api/v1/tenants/ \
      -H "Authorization: Token {your-token}"

    # With pagination and search
    curl -X GET "http://localhost:8000/api/v1/tenants/?page=1&page_size=20&search=acme" \
      -H "Authorization: Token {your-token}"

    # Filter active tenants, ordered by name
    curl -X GET "http://localhost:8000/api/v1/tenants/?is_active=true&ordering=name" \
      -H "Authorization: Token {your-token}"
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/tenants/"
    headers = {"Authorization": "Token {your-token}"}

    # List all tenants
    response = requests.get(url, headers=headers)
    tenants = response.json()

    # With filters
    params = {
        "is_active": True,
        "search": "acme",
        "ordering": "name"
    }
    response = requests.get(url, headers=headers, params=params)
    filtered_tenants = response.json()

    for tenant in filtered_tenants['results']:
        print(f"{tenant['name']} ({tenant['slug']})")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    // List all tenants
    const response = await fetch('http://localhost:8000/api/v1/tenants/', {
      headers: {
        'Authorization': `Token ${token}`
      }
    });

    const data = await response.json();
    console.log('Tenants:', data.results);

    // With query parameters
    const params = new URLSearchParams({
      is_active: 'true',
      search: 'acme',
      ordering: 'name',
      page_size: '20'
    });

    const filteredResponse = await fetch(
      `http://localhost:8000/api/v1/tenants/?${params}`,
      { headers: { 'Authorization': `Token ${token}` } }
    );

    const filtered = await filteredResponse.json();
    ```

=== "TypeScript"
    ```typescript
    interface Tenant {
      id: number;
      uuid: string;
      name: string;
      slug: string;
      email: string;
      url: string;
      description: string;
      is_active: boolean;
      user_count: number;
      created_at: string;
      updated_at: string;
    }

    interface PaginatedResponse<T> {
      count: number;
      next: string | null;
      previous: string | null;
      results: T[];
    }

    async function listTenants(
      params?: {
        page?: number;
        page_size?: number;
        search?: string;
        is_active?: boolean;
        ordering?: string;
      }
    ): Promise<PaginatedResponse<Tenant>> {
      const token = localStorage.getItem('auth_token');
      const queryParams = new URLSearchParams(
        params as Record<string, string>
      );

      const response = await fetch(
        `http://localhost:8000/api/v1/tenants/?${queryParams}`,
        {
          headers: { 'Authorization': `Token ${token}` }
        }
      );

      if (!response.ok) {
        throw new Error('Failed to fetch tenants');
      }

      return response.json();
    }

    // Usage
    const { results, count } = await listTenants({
      is_active: true,
      ordering: 'name'
    });

    console.log(`Found ${count} tenants`);
    results.forEach(tenant => {
      console.log(`- ${tenant.name}`);
    });
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "count": 25,
  "next": "http://localhost:8000/api/v1/tenants/?page=2",
  "previous": null,
  "results": [
    {
      "id": 1,
      "uuid": "550e8400-e29b-41d4-a716-446655440000",
      "name": "ACME Corporation",
      "slug": "acme-corp",
      "email": "admin@acme.com",
      "url": "https://acme.com",
      "description": "Leading provider of innovative solutions",
      "is_active": true,
      "user_count": 15,
      "created_at": "2025-01-15T10:30:00Z",
      "updated_at": "2025-11-30T14:20:00Z"
    },
    {
      "id": 2,
      "uuid": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
      "name": "TechStart Inc",
      "slug": "techstart",
      "email": "contact@techstart.io",
      "url": "https://techstart.io",
      "description": "Startup accelerator",
      "is_active": true,
      "user_count": 8,
      "created_at": "2025-02-01T08:15:00Z",
      "updated_at": "2025-11-29T16:45:00Z"
    }
  ]
}
```

### Error Response

**Code:** `401 Unauthorized`

```json
{
  "detail": "Invalid token."
}
```

---

## Create Tenant

Create a new tenant (organization).

!!! warning "Permissions Required"
    Only SuperAdmin and Admin roles can create tenants.

### Endpoint

```
POST /api/v1/tenants/
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| name | string | ✅ | Tenant name (unique) |
| slug | string | ✅ | URL-friendly identifier (unique) |
| email | string | ❌ | Contact email |
| url | string | ❌ | Website URL |
| description | string | ❌ | Description |
| is_active | boolean | ❌ | Active status (default: true) |

### Example Requests

=== "cURL"
    ```bash
    curl -X POST http://localhost:8000/api/v1/tenants/ \
      -H "Authorization: Token {your-token}" \
      -H "Content-Type: application/json" \
      -d '{
        "name": "New Company LLC",
        "slug": "new-company",
        "email": "info@newcompany.com",
        "url": "https://newcompany.com",
        "description": "A new customer organization",
        "is_active": true
      }'
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/tenants/"
    headers = {
        "Authorization": "Token {your-token}",
        "Content-Type": "application/json"
    }
    payload = {
        "name": "New Company LLC",
        "slug": "new-company",
        "email": "info@newcompany.com",
        "url": "https://newcompany.com",
        "description": "A new customer organization",
        "is_active": True
    }

    response = requests.post(url, headers=headers, json=payload)
    tenant = response.json()
    print(f"Created tenant: {tenant['name']} (ID: {tenant['id']})")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    const response = await fetch('http://localhost:8000/api/v1/tenants/', {
      method: 'POST',
      headers: {
        'Authorization': `Token ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        name: 'New Company LLC',
        slug: 'new-company',
        email: 'info@newcompany.com',
        url: 'https://newcompany.com',
        description: 'A new customer organization',
        is_active: true
      })
    });

    if (response.ok) {
      const tenant = await response.json();
      console.log('Tenant created:', tenant);
    } else {
      const error = await response.json();
      console.error('Error:', error);
    }
    ```

### Success Response

**Code:** `201 Created`

```json
{
  "id": 10,
  "uuid": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "name": "New Company LLC",
  "slug": "new-company",
  "email": "info@newcompany.com",
  "url": "https://newcompany.com",
  "description": "A new customer organization",
  "is_active": true,
  "user_count": 0,
  "created_at": "2025-11-30T15:30:00Z",
  "updated_at": "2025-11-30T15:30:00Z"
}
```

### Error Responses

**Code:** `400 Bad Request` (Slug already exists)

```json
{
  "slug": ["tenant with this slug already exists."]
}
```

**Code:** `400 Bad Request` (Invalid slug format)

```json
{
  "slug": ["Enter a valid 'slug' consisting of letters, numbers, underscores or hyphens."]
}
```

**Code:** `403 Forbidden` (Insufficient permissions)

```json
{
  "detail": "You do not have permission to perform this action."
}
```

---

## Retrieve Tenant

Get details of a specific tenant.

### Endpoint

```
GET /api/v1/tenants/{id}/
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| id | integer | Tenant ID |

### Example Requests

=== "cURL"
    ```bash
    curl -X GET http://localhost:8000/api/v1/tenants/1/ \
      -H "Authorization: Token {your-token}"
    ```

=== "Python"
    ```python
    import requests

    tenant_id = 1
    url = f"http://localhost:8000/api/v1/tenants/{tenant_id}/"
    headers = {"Authorization": "Token {your-token}"}

    response = requests.get(url, headers=headers)
    tenant = response.json()
    print(f"Tenant: {tenant['name']}")
    print(f"Users: {tenant['user_count']}")
    print(f"Status: {'Active' if tenant['is_active'] else 'Inactive'}")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');
    const tenantId = 1;

    const response = await fetch(
      `http://localhost:8000/api/v1/tenants/${tenantId}/`,
      {
        headers: { 'Authorization': `Token ${token}` }
      }
    );

    const tenant = await response.json();
    console.log('Tenant details:', tenant);
    ```

### Success Response

**Code:** `200 OK`

```json
{
  "id": 1,
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "name": "ACME Corporation",
  "slug": "acme-corp",
  "email": "admin@acme.com",
  "url": "https://acme.com",
  "description": "Leading provider of innovative solutions",
  "is_active": true,
  "user_count": 15,
  "created_at": "2025-01-15T10:30:00Z",
  "updated_at": "2025-11-30T14:20:00Z"
}
```

### Error Responses

**Code:** `404 Not Found`

```json
{
  "detail": "Not found."
}
```

**Code:** `403 Forbidden` (Accessing other tenant as Tenant Owner)

```json
{
  "detail": "You do not have permission to perform this action."
}
```

---

## Update Tenant

Update an existing tenant.

### Endpoint

```
PUT /api/v1/tenants/{id}/     # Full update
PATCH /api/v1/tenants/{id}/   # Partial update
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| id | integer | Tenant ID |

### Request Body

All fields optional for PATCH:

| Field | Type | Description |
|-------|------|-------------|
| name | string | Tenant name |
| slug | string | URL identifier |
| email | string | Contact email |
| url | string | Website URL |
| description | string | Description |
| is_active | boolean | Active status |

!!! warning "Read-Only Fields"
    Cannot update: `id`, `uuid`, `created_at`, `updated_at`, `user_count`

### Example Requests

=== "cURL"
    ```bash
    curl -X PATCH http://localhost:8000/api/v1/tenants/1/ \
      -H "Authorization: Token {your-token}" \
      -H "Content-Type: application/json" \
      -d '{
        "description": "Updated description",
        "url": "https://new-website.com"
      }'
    ```

=== "Python"
    ```python
    import requests

    tenant_id = 1
    url = f"http://localhost:8000/api/v1/tenants/{tenant_id}/"
    headers = {
        "Authorization": "Token {your-token}",
        "Content-Type": "application/json"
    }
    payload = {
        "description": "Updated description",
        "url": "https://new-website.com"
    }

    response = requests.patch(url, headers=headers, json=payload)
    updated_tenant = response.json()
    print(f"Updated: {updated_tenant['name']}")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');
    const tenantId = 1;

    const response = await fetch(
      `http://localhost:8000/api/v1/tenants/${tenantId}/`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Token ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          description: 'Updated description',
          url: 'https://new-website.com'
        })
      }
    );

    const tenant = await response.json();
    console.log('Updated tenant:', tenant);
    ```

### Success Response

**Code:** `200 OK`

Returns the updated tenant object (same format as GET).

### Error Responses

**Code:** `400 Bad Request` (Duplicate slug)

```json
{
  "slug": ["tenant with this slug already exists."]
}
```

**Code:** `403 Forbidden`

```json
{
  "detail": "You do not have permission to perform this action."
}
```

---

## Delete Tenant

Delete a tenant.

!!! danger "Destructive Operation"
    This operation cannot be undone. All associated data will be lost.

### Endpoint

```
DELETE /api/v1/tenants/{id}/
```

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| id | integer | Tenant ID |

### Example Requests

=== "cURL"
    ```bash
    curl -X DELETE http://localhost:8000/api/v1/tenants/10/ \
      -H "Authorization: Token {your-token}"
    ```

=== "Python"
    ```python
    import requests

    tenant_id = 10
    url = f"http://localhost:8000/api/v1/tenants/{tenant_id}/"
    headers = {"Authorization": "Token {your-token}"}

    response = requests.delete(url, headers=headers)
    if response.status_code == 204:
        print(f"Tenant {tenant_id} deleted successfully")
    else:
        print(f"Error: {response.json()}")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');
    const tenantId = 10;

    const response = await fetch(
      `http://localhost:8000/api/v1/tenants/${tenantId}/`,
      {
        method: 'DELETE',
        headers: { 'Authorization': `Token ${token}` }
      }
    );

    if (response.ok) {
      console.log('Tenant deleted successfully');
    } else {
      const error = await response.json();
      console.error('Delete failed:', error);
    }
    ```

### Success Response

**Code:** `204 No Content`

(No response body)

### Error Responses

**Code:** `403 Forbidden`

```json
{
  "detail": "You do not have permission to perform this action."
}
```

**Code:** `400 Bad Request` (Tenant has users)

```json
{
  "detail": "Cannot delete tenant with existing users."
}
```

!!! info "Delete Protection"
    Tenants with associated users cannot be deleted due to PROTECT constraint.
    Delete or reassign all users first.

---

## Get Current User's Tenant

Retrieve the authenticated user's tenant.

### Endpoint

```
GET /api/v1/tenants/me/
```

### Example Requests

=== "cURL"
    ```bash
    curl -X GET http://localhost:8000/api/v1/tenants/me/ \
      -H "Authorization: Token {your-token}"
    ```

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/tenants/me/"
    headers = {"Authorization": "Token {your-token}"}

    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        tenant = response.json()
        print(f"My tenant: {tenant['name']}")
    else:
        print("No tenant assigned")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    const response = await fetch('http://localhost:8000/api/v1/tenants/me/', {
      headers: { 'Authorization': `Token ${token}` }
    });

    if (response.ok) {
      const tenant = await response.json();
      console.log('My tenant:', tenant);
    } else {
      console.log('No tenant assigned');
    }
    ```

### Success Response

**Code:** `200 OK`

Returns tenant object (same format as GET `/tenants/{id}/`).

### Error Response

**Code:** `404 Not Found` (User has no tenant)

```json
{
  "detail": "No tenant assigned"
}
```

---

## Batch Operations

### Create Multiple Tenants

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/tenants/"
    headers = {
        "Authorization": "Token {your-token}",
        "Content-Type": "application/json"
    }

    tenants_to_create = [
        {
            "name": "Company A",
            "slug": "company-a",
            "email": "info@company-a.com"
        },
        {
            "name": "Company B",
            "slug": "company-b",
            "email": "info@company-b.com"
        },
        {
            "name": "Company C",
            "slug": "company-c",
            "email": "info@company-c.com"
        }
    ]

    created_tenants = []
    for tenant_data in tenants_to_create:
        response = requests.post(url, headers=headers, json=tenant_data)
        if response.status_code == 201:
            created_tenants.append(response.json())
            print(f"✓ Created: {tenant_data['name']}")
        else:
            print(f"✗ Failed: {tenant_data['name']} - {response.json()}")

    print(f"\nCreated {len(created_tenants)} tenants")
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    const tenantsToCreate = [
      { name: 'Company A', slug: 'company-a', email: 'info@company-a.com' },
      { name: 'Company B', slug: 'company-b', email: 'info@company-b.com' },
      { name: 'Company C', slug: 'company-c', email: 'info@company-c.com' }
    ];

    const createTenant = async (tenantData) => {
      const response = await fetch('http://localhost:8000/api/v1/tenants/', {
        method: 'POST',
        headers: {
          'Authorization': `Token ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(tenantData)
      });
      return response.json();
    };

    // Create all tenants in parallel
    const results = await Promise.allSettled(
      tenantsToCreate.map(createTenant)
    );

    results.forEach((result, index) => {
      if (result.status === 'fulfilled') {
        console.log(`✓ Created: ${tenantsToCreate[index].name}`);
      } else {
        console.log(`✗ Failed: ${tenantsToCreate[index].name}`);
      }
    });
    ```

## Filtering and Search

### Advanced Filtering Examples

=== "Python"
    ```python
    import requests

    url = "http://localhost:8000/api/v1/tenants/"
    headers = {"Authorization": "Token {your-token}"}

    # Active tenants only
    response = requests.get(url, headers=headers, params={"is_active": True})

    # Search by name or email
    response = requests.get(url, headers=headers, params={"search": "tech"})

    # Order by most recent
    response = requests.get(url, headers=headers, params={"ordering": "-created_at"})

    # Combined filters
    params = {
        "is_active": True,
        "search": "startup",
        "ordering": "name",
        "page_size": 50
    }
    response = requests.get(url, headers=headers, params=params)
    ```

=== "JavaScript"
    ```javascript
    const token = localStorage.getItem('auth_token');

    async function searchTenants(filters) {
      const params = new URLSearchParams(filters);
      const response = await fetch(
        `http://localhost:8000/api/v1/tenants/?${params}`,
        { headers: { 'Authorization': `Token ${token}` } }
      );
      return response.json();
    }

    // Active tenants only
    const active = await searchTenants({ is_active: 'true' });

    // Search by keyword
    const searched = await searchTenants({ search: 'tech' });

    // Most recent first
    const recent = await searchTenants({ ordering: '-created_at' });

    // Combined
    const filtered = await searchTenants({
      is_active: 'true',
      search: 'startup',
      ordering: 'name',
      page_size: '50'
    });
    ```

## Common Use Cases

### Use Case 1: Onboard New Customer

```python
import requests

def onboard_customer(name, contact_email, admin_username, admin_email):
    """Complete customer onboarding workflow."""
    base_url = "http://localhost:8000/api/v1"
    headers = {
        "Authorization": "Token {your-admin-token}",
        "Content-Type": "application/json"
    }

    # Step 1: Create tenant
    tenant_data = {
        "name": name,
        "slug": name.lower().replace(" ", "-"),
        "email": contact_email
    }
    response = requests.post(
        f"{base_url}/tenants/",
        headers=headers,
        json=tenant_data
    )
    tenant = response.json()
    print(f"✓ Created tenant: {tenant['name']}")

    # Step 2: Create tenant owner
    user_data = {
        "username": admin_username,
        "email": admin_email,
        "role": "tenant_owner",
        "tenant": tenant['id'],
        "password": "TemporaryPassword123!"  # User should change
    }
    response = requests.post(
        f"{base_url}/users/",
        headers=headers,
        json=user_data
    )
    user = response.json()
    print(f"✓ Created tenant owner: {user['username']}")

    return {
        "tenant": tenant,
        "admin_user": user
    }

# Usage
result = onboard_customer(
    name="New Corp Inc",
    contact_email="contact@newcorp.com",
    admin_username="admin@newcorp",
    admin_email="admin@newcorp.com"
)
```

### Use Case 2: Deactivate Tenant

```python
def deactivate_tenant(tenant_id):
    """Deactivate a tenant and all its users."""
    base_url = "http://localhost:8000/api/v1"
    headers = {
        "Authorization": "Token {your-admin-token}",
        "Content-Type": "application/json"
    }

    # Deactivate tenant
    response = requests.patch(
        f"{base_url}/tenants/{tenant_id}/",
        headers=headers,
        json={"is_active": False}
    )
    tenant = response.json()
    print(f"✓ Deactivated tenant: {tenant['name']}")

    # Optionally deactivate all users
    response = requests.get(
        f"{base_url}/users/?tenant={tenant_id}",
        headers=headers
    )
    users = response.json()['results']

    for user in users:
        requests.patch(
            f"{base_url}/users/{user['id']}/",
            headers=headers,
            json={"is_active": False}
        )
        print(f"  ✓ Deactivated user: {user['username']}")

    print(f"Deactivated {len(users)} users")
```

## Next Steps

- **[Users API](users.md)** - User management endpoints
- **[Subscriptions API](subscriptions.md)** - Billing and subscriptions
- **[User Guide](../user-guide/organizations/tenants.md)** - Managing tenants in the UI

---

!!! question "Need Help?"
    For API questions, check the [FAQ](../troubleshooting/faq.md) or contact dev@thinesoft.com
