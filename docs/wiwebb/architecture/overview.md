# Architecture Overview

This document provides a comprehensive overview of WiWebb's system architecture, including its components, data flow, and design decisions.

## System Architecture

WiWebb is built on a modern, containerized microservices architecture designed for scalability, maintainability, and multi-tenancy.

### High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        WebApp[Web Application<br/>React 19 + TypeScript]
        MobileApp[Mobile Application<br/>Coming Soon]
    end

    subgraph "Load Balancer / Proxy"
        Nginx[Nginx<br/>Reverse Proxy]
    end

    subgraph "Application Layer"
        Frontend[Frontend Server<br/>Vite Dev Server<br/>Port 5173]
        API[Django REST API<br/>Gunicorn + Django 5.2<br/>Port 8000]
    end

    subgraph "Business Logic Layer"
        Auth[Authentication<br/>dj-rest-auth + Allauth]
        MultiTenant[Multi-Tenancy<br/>Tenant Isolation]
        Network[Network Services<br/>RADIUS, VPN, IPAM]
        Billing[Billing & Subscriptions<br/>Stripe Integration]
        CMS[Content Management<br/>Django CMS]
    end

    subgraph "Data Layer"
        Postgres[(PostgreSQL<br/>Primary Database)]
        Redis[(Redis<br/>Cache & Queue)]
    end

    subgraph "External Services"
        Stripe[Stripe API<br/>Payment Gateway]
        LocalStripe[LocalStripe<br/>Dev Mock Server]
    end

    subgraph "Background Processing"
        Celery[Celery Workers<br/>Async Tasks]
    end

    WebApp --> Nginx
    MobileApp -.-> Nginx
    Nginx --> Frontend
    Nginx --> API
    Frontend --> API

    API --> Auth
    API --> MultiTenant
    API --> Network
    API --> Billing
    API --> CMS

    Auth --> Postgres
    MultiTenant --> Postgres
    Network --> Postgres
    Billing --> Postgres
    CMS --> Postgres

    API --> Redis
    Celery --> Redis
    Celery --> Postgres

    Billing --> Stripe
    Billing -.-> LocalStripe

    style WebApp fill:#e3f2fd
    style Frontend fill:#fff3e0
    style API fill:#f3e5f5
    style Postgres fill:#e8f5e9
    style MobileApp stroke-dasharray: 5 5
```

## Technology Stack

### Frontend

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| Framework | React | 19.1.1 | UI library |
| Language | TypeScript | Latest | Type-safe development |
| Build Tool | Vite | 7.1.2 | Fast bundling |
| State Management | TanStack Query | 5.85.5 | Server state management |
| Routing | React Router | 7.8.1 | Client-side routing |
| UI Components | shadcn/ui | Latest | Component library |
| Styling | Tailwind CSS | 4.1.12 | Utility-first CSS |
| Charts | Recharts | Latest | Data visualization |
| Forms | React Hook Form + Zod | Latest | Form validation |

### Backend

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| Framework | Django | 5.2.3 | Web framework |
| API Framework | Django REST Framework | Latest | RESTful APIs |
| Language | Python | 3.11+ | Backend language |
| Database | PostgreSQL | 14+ | Primary database |
| Cache/Queue | Redis | 7+ | Caching & task queue |
| Task Queue | Celery | Latest | Async task processing |
| Auth | dj-rest-auth + Allauth | Latest | Authentication |
| Payments | dj-stripe + django-payments | Latest | Payment processing |
| CMS | Django CMS | 5.0.1 | Content management |

### Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| Containerization | Docker | Application packaging |
| Orchestration | Docker Compose | Multi-container deployment |
| Web Server | Nginx | Reverse proxy (production) |
| WSGI Server | Gunicorn | Python application server |
| Payment Dev | LocalStripe | Stripe mock server |

## Container Architecture

WiWebb runs as multiple Docker containers orchestrated by Docker Compose:

```mermaid
graph LR
    subgraph "Docker Network: wiwebb_network"
        Django[django<br/>Django API<br/>:8000]
        Frontend[frontend<br/>React App<br/>:5173]
        DB[(django_db<br/>PostgreSQL<br/>:5432)]
        Redis[(redis<br/>Redis<br/>:6379)]
        LocalStripe[localstripe<br/>Stripe Mock<br/>:8420]
        Celery[celery<br/>Worker]
    end

    Frontend --> Django
    Django --> DB
    Django --> Redis
    Django --> LocalStripe
    Celery --> Redis
    Celery --> DB

    style Django fill:#f3e5f5
    style Frontend fill:#fff3e0
    style DB fill:#e8f5e9
    style Redis fill:#fce4ec
```

### Container Details

#### django
- **Image:** Custom (built from Dockerfile)
- **Purpose:** Django REST API server
- **Exposed Ports:** 8000
- **Volumes:**
    - `./backend:/app` (source code)
    - `static_volume:/app/staticfiles` (static files)
    - `media_volume:/app/mediafiles` (media uploads)
- **Dependencies:** django_db, redis

#### frontend
- **Image:** Custom (built from Dockerfile)
- **Purpose:** React development server
- **Exposed Ports:** 5173
- **Volumes:** `./frontend:/app` (source code)
- **Environment:** `VITE_API_URL` for backend connection

#### django_db
- **Image:** postgres:14-alpine
- **Purpose:** Primary PostgreSQL database
- **Exposed Ports:** 5432 (internal only)
- **Volumes:** `postgres_data:/var/lib/postgresql/data`
- **Environment:** Database credentials from `.env`

#### redis
- **Image:** redis:7-alpine
- **Purpose:** Cache and Celery broker
- **Exposed Ports:** 6379 (internal only)
- **Volumes:** `redis_data:/data`

#### localstripe
- **Image:** stripe/stripe-mock
- **Purpose:** Mock Stripe API for development
- **Exposed Ports:** 8420
- **Condition:** Only in development (`USE_LOCALSTRIPE=True`)

#### celery
- **Image:** Same as django
- **Purpose:** Background task worker
- **Command:** `celery -A backend worker --loglevel=info`
- **Dependencies:** redis, django_db

## Data Flow Architecture

### Request Flow

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant Nginx
    participant Frontend
    participant API
    participant Auth
    participant DB
    participant Redis

    User->>Browser: Access application
    Browser->>Nginx: HTTP Request
    Nginx->>Frontend: Route to React app
    Frontend-->>Browser: HTML/JS/CSS
    Browser->>Browser: Render UI

    User->>Browser: Perform action
    Browser->>API: API Request + Token
    API->>Redis: Check cache
    alt Cache hit
        Redis-->>API: Cached data
    else Cache miss
        API->>Auth: Verify token
        Auth->>DB: Query user/tenant
        DB-->>Auth: User data
        Auth-->>API: Authenticated
        API->>DB: Query data
        DB-->>API: Results
        API->>Redis: Cache results
    end
    API-->>Browser: JSON response
    Browser-->>User: Update UI
```

### Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant Auth
    participant DB

    User->>Frontend: Enter credentials
    Frontend->>API: POST /auth/login/
    API->>Auth: Authenticate user
    Auth->>DB: Verify credentials
    DB-->>Auth: User found
    Auth->>DB: Create auth token
    DB-->>Auth: Token created
    Auth-->>API: Token + user data
    API-->>Frontend: {key: "token...", user: {...}}
    Frontend->>Frontend: Store token
    Frontend->>API: GET /auth/user/ (with token)
    API->>Auth: Verify token
    Auth->>DB: Get user details
    DB-->>Auth: Full user data
    Auth-->>API: User object
    API-->>Frontend: User profile
    Frontend-->>User: Redirect to dashboard
```

### Payment Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant Billing
    participant Stripe
    participant Webhook

    User->>Frontend: Select plan
    Frontend->>API: POST /subscriptions/subscribe/
    API->>Billing: Create subscription
    Billing->>Stripe: Create customer (if needed)
    Stripe-->>Billing: Customer ID
    Billing->>Stripe: Create subscription
    Stripe-->>Billing: Subscription details
    Billing->>DB: Save subscription
    Billing-->>API: Subscription status
    API-->>Frontend: Response
    Frontend-->>User: Show confirmation

    Note over Stripe,Webhook: Async webhook

    Stripe->>API: POST /payments/webhook/
    API->>Webhook: Process event
    Webhook->>DB: Update payment status
    Webhook->>DB: Log event
    Webhook-->>Stripe: 200 OK
```

## Application Layers

### Presentation Layer

**Responsibility:** User interface and user experience

**Components:**
- React components (pages, layouts, forms)
- Routing (React Router)
- State management (TanStack Query, Context)
- UI components (shadcn/ui)

**Key Directories:**
```
frontend/src/
├── pages/          # Route components
├── components/     # Reusable UI components
├── layouts/        # Layout components
├── hooks/          # Custom React hooks
└── context/        # React Context providers
```

### API Layer

**Responsibility:** HTTP endpoints and request/response handling

**Components:**
- Django REST Framework views
- Serializers for data transformation
- URL routing
- Permission classes

**Key Directories:**
```
backend/
├── api/views.py              # User/Tenant API views
├── api/serializers.py        # Data serialization
├── payment_gateway/views.py  # Payment API views
└── payment_gateway/serializers.py
```

### Business Logic Layer

**Responsibility:** Core application logic and business rules

**Components:**
- Django models
- Model managers and querysets
- Business logic methods
- Service classes

**Key Files:**
- `api/models.py` - User, Tenant, Todo, AuditLog
- `payment_gateway/models.py` - Plans, Payments, Subscriptions

### Data Access Layer

**Responsibility:** Database interactions and data persistence

**Components:**
- Django ORM
- Model managers
- Database migrations
- Indexing strategies

## Multi-Tenancy Architecture

WiWebb implements **shared database with tenant isolation** pattern:

```mermaid
graph TB
    Request[API Request] --> Middleware[Tenant Middleware]
    Middleware --> Auth[Get User]
    Auth --> Tenant[Get User.tenant]
    Tenant --> Filter[Apply Tenant Filter]
    Filter --> QuerySet[Filtered QuerySet]

    subgraph "Database"
        AllData[(All Tenant Data)]
        Tenant1[Tenant 1 Data]
        Tenant2[Tenant 2 Data]
        Tenant3[Tenant 3 Data]
    end

    QuerySet --> Tenant1
    QuerySet -.-> Tenant2
    QuerySet -.-> Tenant3

    style Tenant1 fill:#e8f5e9
    style Tenant2 fill:#e0e0e0,opacity:0.3
    style Tenant3 fill:#e0e0e0,opacity:0.3
```

### Tenant Isolation Mechanism

**Queryset Scoping:**
```python
# Automatic tenant filtering in views
class TenantViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        user = self.request.user
        if user.role in ["superadmin", "admin"]:
            # See all tenants
            return Tenant.objects.all()
        else:
            # See only own tenant
            return Tenant.objects.filter(id=user.tenant_id)
```

**Benefits:**
- Single database reduces infrastructure complexity
- Data isolation enforced at application level
- Cost-effective for scaling
- Easier backups and maintenance

**Security:**
- Tenant ID in all queries
- Row-level security
- Audit logging for all sensitive actions

## Scalability Design

### Horizontal Scaling

```mermaid
graph TB
    LB[Load Balancer<br/>Nginx/HAProxy]

    subgraph "API Tier - Horizontally Scaled"
        API1[Django API 1]
        API2[Django API 2]
        API3[Django API 3]
    end

    subgraph "Worker Tier - Horizontally Scaled"
        Worker1[Celery Worker 1]
        Worker2[Celery Worker 2]
        Worker3[Celery Worker 3]
    end

    subgraph "Data Tier"
        DBMaster[(PostgreSQL<br/>Primary)]
        DBReplica1[(PostgreSQL<br/>Replica 1)]
        DBReplica2[(PostgreSQL<br/>Replica 2)]
        Redis[(Redis<br/>Cluster)]
    end

    LB --> API1
    LB --> API2
    LB --> API3

    API1 --> DBMaster
    API2 --> DBMaster
    API3 --> DBMaster

    API1 -.->|Read| DBReplica1
    API2 -.->|Read| DBReplica2

    API1 --> Redis
    API2 --> Redis
    API3 --> Redis

    Worker1 --> Redis
    Worker2 --> Redis
    Worker3 --> Redis

    Worker1 --> DBMaster
    Worker2 --> DBMaster
    Worker3 --> DBMaster

    DBMaster -->|Replication| DBReplica1
    DBMaster -->|Replication| DBReplica2
```

### Caching Strategy

**Redis Caching:**
- Session data
- User authentication tokens
- API response caching
- Database query result caching

**Cache Invalidation:**
- Time-based expiration
- Event-based invalidation
- Manual cache clearing

## Security Architecture

### Defense in Depth

```mermaid
graph TB
    User[User Request]

    User --> Layer1[1. Network Security<br/>HTTPS, Firewall]
    Layer1 --> Layer2[2. Authentication<br/>Token-based]
    Layer2 --> Layer3[3. Authorization<br/>RBAC, Tenant Scope]
    Layer3 --> Layer4[4. Input Validation<br/>Serializers]
    Layer4 --> Layer5[5. Data Encryption<br/>DB encryption at rest]
    Layer5 --> Layer6[6. Audit Logging<br/>All sensitive actions]
    Layer6 --> Resource[Protected Resource]

    style Layer1 fill:#ffebee
    style Layer2 fill:#fff3e0
    style Layer3 fill:#f3e5f5
    style Layer4 fill:#e8f5e9
    style Layer5 fill:#e3f2fd
    style Layer6 fill:#fce4ec
```

### Security Features

1. **Authentication:** Token-based (django-rest-auth)
2. **Authorization:** Role-based access control (RBAC)
3. **Multi-tenancy:** Tenant-scoped data access
4. **Input Validation:** Django REST Framework serializers
5. **SQL Injection:** Django ORM parameterized queries
6. **XSS Protection:** Django templates auto-escaping
7. **CSRF Protection:** Django CSRF middleware
8. **Audit Logging:** All user/tenant modifications logged

## Performance Optimization

### Database Optimization

**Indexing Strategy:**
```python
class User(AbstractUser):
    tenant = models.ForeignKey(
        Tenant,
        db_index=True,  # Indexed for fast filtering
        ...
    )
    role = models.CharField(
        db_index=True,  # Indexed for permission checks
        ...
    )
```

**Query Optimization:**
- Select related / prefetch related for joins
- Queryset caching with Redis
- Database connection pooling
- Read replicas for scaling

### API Optimization

- Pagination for large result sets
- Field filtering in serializers
- Response compression
- HTTP caching headers

## Deployment Architecture

### Production Deployment

```mermaid
graph TB
    Internet[Internet]

    Internet --> CDN[CDN<br/>Static Assets]
    Internet --> LB[Load Balancer]

    LB --> Nginx1[Nginx 1]
    LB --> Nginx2[Nginx 2]

    Nginx1 --> API1[Django API 1]
    Nginx1 --> API2[Django API 2]
    Nginx2 --> API3[Django API 3]

    API1 --> DB[(PostgreSQL<br/>RDS/Managed)]
    API2 --> DB
    API3 --> DB

    API1 --> Redis[(Redis<br/>ElastiCache)]
    API2 --> Redis
    API3 --> Redis

    API1 --> S3[(S3<br/>Media Storage)]
    API2 --> S3
    API3 --> S3

    Worker1[Celery Worker 1] --> Redis
    Worker2[Celery Worker 2] --> Redis
    Worker1 --> DB
    Worker2 --> DB
```

## Monitoring and Observability

### Logging Architecture

```mermaid
graph LR
    App1[Django API 1] --> Logs1[Application Logs]
    App2[Django API 2] --> Logs2[Application Logs]
    Worker[Celery] --> Logs3[Worker Logs]

    Logs1 --> Aggregator[Log Aggregator<br/>ELK/CloudWatch]
    Logs2 --> Aggregator
    Logs3 --> Aggregator

    Aggregator --> Dashboard[Dashboard<br/>Kibana/Grafana]
    Aggregator --> Alerts[Alert System<br/>PagerDuty]
```

### Metrics Collection

- **Application Metrics:** Request rate, latency, errors
- **System Metrics:** CPU, memory, disk usage
- **Database Metrics:** Query performance, connection pool
- **Business Metrics:** User signups, subscriptions, payments

## Design Principles

### SOLID Principles

- **Single Responsibility:** Each model/view has one clear purpose
- **Open/Closed:** Extensible via Django apps
- **Liskov Substitution:** Proper use of base classes
- **Interface Segregation:** Focused serializers and viewsets
- **Dependency Inversion:** Dependency injection via Django

### Django Best Practices

- **Fat Models, Thin Views:** Business logic in models
- **DRY (Don't Repeat Yourself):** Shared utilities and mixins
- **12-Factor App:** Environment-based configuration
- **RESTful API Design:** Standard HTTP methods and status codes

## Next Steps

Dive deeper into specific architectural components:

- **[Backend Architecture](backend.md)** - Django app structure and design
- **[Frontend Architecture](frontend.md)** - React app structure and patterns
- **[Database Schema](database-schema.md)** - Entity relationships and models
- **[Authentication](authentication.md)** - Auth flow and security
- **[Multi-Tenancy](multi-tenancy.md)** - Tenant isolation details

---

!!! info "Questions?"
    For architectural questions or suggestions, contact the development team at dev@thinesoft.com
