# Envoy Fleet Management System - API Specifications

## 1. Overview

This document defines the RESTful API specifications for the Envoy Fleet Management System. All APIs are built using Axum framework in Rust and follow REST principles with JSON payloads.

## 2. API Standards

### 2.1 Base URL Structure
```
https://api.envoy-control.example.com/api/v1
```

### 2.2 Authentication
All API endpoints require JWT authentication except for `/auth` endpoints.

```http
Authorization: Bearer <JWT_TOKEN>
```

### 2.3 Request/Response Format
- **Content-Type**: `application/json`
- **Accept**: `application/json`
- **Timestamp Format**: ISO 8601 (`2024-01-15T10:30:00Z`)
- **UUID Format**: Standard UUID v4

### 2.4 Standard HTTP Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET, PUT |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid request body |
| 401 | Unauthorized | Invalid/missing authentication |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Resource conflict |
| 422 | Unprocessable Entity | Validation error |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |

### 2.5 Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": {
      "field": "email",
      "reason": "Invalid email format"
    },
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

## 3. Authentication API

### 3.1 Login

```http
POST /api/v1/auth/login
```

**Request:**
```json
{
  "email": "user@example.com",
  "password": "secure_password"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "username": "johndoe",
    "role": "operator",
    "tenant_id": "660e8400-e29b-41d4-a716-446655440001"
  }
}
```

### 3.2 Refresh Token

```http
POST /api/v1/auth/refresh
```

**Request:**
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

### 3.3 Logout

```http
POST /api/v1/auth/logout
```

## 4. Virtual Machine API

### 4.1 List Virtual Machines

```http
GET /api/v1/virtual-machines
```

**Query Parameters:**
- `page` (integer): Page number (default: 1)
- `per_page` (integer): Items per page (default: 20, max: 100)
- `status` (string): Filter by status
- `region` (string): Filter by region
- `sort_by` (string): Sort field (hostname, created_at)
- `order` (string): Sort order (asc, desc)

**Response:**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "hostname": "vm-prod-001",
      "ip_address": "10.0.1.5",
      "region": "us-west-2",
      "availability_zone": "us-west-2a",
      "status": "active",
      "cpu_cores": 8,
      "memory_mb": 16384,
      "disk_gb": 500,
      "container_count": 5,
      "last_heartbeat": "2024-01-15T10:25:00Z",
      "metadata": {
        "provider": "aws",
        "instance_type": "t3.xlarge"
      },
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-15T10:25:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total_pages": 5,
    "total_items": 95
  }
}
```

### 4.2 Get Virtual Machine

```http
GET /api/v1/virtual-machines/{id}
```

### 4.3 Create Virtual Machine

```http
POST /api/v1/virtual-machines
```

**Request:**
```json
{
  "hostname": "vm-prod-002",
  "ip_address": "10.0.1.6",
  "region": "us-west-2",
  "availability_zone": "us-west-2b",
  "cpu_cores": 8,
  "memory_mb": 16384,
  "disk_gb": 500,
  "metadata": {
    "provider": "aws",
    "instance_type": "t3.xlarge"
  }
}
```

### 4.4 Update Virtual Machine

```http
PUT /api/v1/virtual-machines/{id}
```

### 4.5 Delete Virtual Machine

```http
DELETE /api/v1/virtual-machines/{id}
```

### 4.6 Get VM Containers

```http
GET /api/v1/virtual-machines/{id}/containers
```

## 5. Container API

### 5.1 List Containers

```http
GET /api/v1/containers
```

### 5.2 Get Container

```http
GET /api/v1/containers/{id}
```

### 5.3 Create Container

```http
POST /api/v1/containers
```

**Request:**
```json
{
  "vm_id": "550e8400-e29b-41d4-a716-446655440000",
  "container_id": "abc123def456",
  "name": "envoy-proxy-01",
  "image": "envoyproxy/envoy:v1.28.0",
  "resource_limits": {
    "cpu": "2",
    "memory": "4Gi"
  },
  "environment_vars": {
    "ENVOY_ADMIN_PORT": "9901",
    "ENVOY_LOG_LEVEL": "info"
  }
}
```

### 5.4 Update Container Status

```http
PATCH /api/v1/containers/{id}/status
```

**Request:**
```json
{
  "status": "running",
  "restart_count": 0
}
```

## 6. Envoy Proxy API

### 6.1 List Envoy Proxies

```http
GET /api/v1/proxies
```

**Response:**
```json
{
  "data": [
    {
      "id": "770e8400-e29b-41d4-a716-446655440000",
      "proxy_id": "proxy-west-001",
      "container_id": "880e8400-e29b-41d4-a716-446655440000",
      "cluster_id": "990e8400-e29b-41d4-a716-446655440000",
      "version": "1.28.0",
      "status": "healthy",
      "admin_address": "10.0.1.5",
      "admin_port": 9901,
      "configuration": {
        "listeners": 3,
        "clusters": 5,
        "routes": 15
      },
      "metrics": {
        "requests_per_second": 1250.5,
        "error_rate": 0.01,
        "p99_latency_ms": 45.2
      },
      "last_heartbeat": "2024-01-15T10:29:00Z",
      "created_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

### 6.2 Get Proxy Configuration

```http
GET /api/v1/proxies/{id}/configuration
```

**Response:**
```json
{
  "id": "aa0e8400-e29b-41d4-a716-446655440000",
  "proxy_id": "770e8400-e29b-41d4-a716-446655440000",
  "name": "proxy-west-001-config",
  "active_version": {
    "id": "bb0e8400-e29b-41d4-a716-446655440000",
    "version_number": 5,
    "created_by": "user@example.com",
    "created_at": "2024-01-15T09:00:00Z"
  },
  "static_resources": {
    "listeners": [...],
    "clusters": [...]
  },
  "dynamic_resources": {
    "lds_config": {...},
    "cds_config": {...},
    "ads_config": {...}
  },
  "admin_config": {
    "address": {
      "socket_address": {
        "address": "0.0.0.0",
        "port_value": 9901
      }
    }
  }
}
```

### 6.3 Update Proxy Configuration

```http
PUT /api/v1/proxies/{id}/configuration
```

**Request:**
```json
{
  "change_description": "Add new upstream cluster for service B",
  "static_resources": {
    "clusters": [
      {
        "name": "service_b",
        "type": "STRICT_DNS",
        "lb_policy": "ROUND_ROBIN",
        "load_assignment": {
          "cluster_name": "service_b",
          "endpoints": [
            {
              "lb_endpoints": [
                {
                  "endpoint": {
                    "address": {
                      "socket_address": {
                        "address": "service-b.internal",
                        "port_value": 8080
                      }
                    }
                  }
                }
              ]
            }
          ]
        }
      }
    ]
  }
}
```

### 6.4 Rollback Configuration

```http
POST /api/v1/proxies/{id}/configuration/rollback
```

**Request:**
```json
{
  "target_version": 3,
  "reason": "Performance degradation detected"
}
```

## 7. Cluster API

### 7.1 List Clusters

```http
GET /api/v1/clusters
```

### 7.2 Create Cluster

```http
POST /api/v1/clusters
```

**Request:**
```json
{
  "name": "service-a-cluster",
  "type": "STRICT_DNS",
  "lb_policy": {
    "type": "ROUND_ROBIN"
  },
  "circuit_breaker": {
    "max_connections": 1000,
    "max_requests": 1000,
    "max_retries": 3
  },
  "outlier_detection": {
    "consecutive_5xx": 5,
    "interval_ms": 30000,
    "eject_percent": 50
  },
  "health_checks": [
    {
      "timeout_ms": 3000,
      "interval_ms": 10000,
      "path": "/health",
      "healthy_threshold": 2,
      "unhealthy_threshold": 3
    }
  ]
}
```

### 7.3 Update Cluster

```http
PUT /api/v1/clusters/{id}
```

### 7.4 Add Cluster Endpoint

```http
POST /api/v1/clusters/{id}/endpoints
```

**Request:**
```json
{
  "address": "10.0.2.10",
  "port": 8080,
  "weight": 100,
  "metadata": {
    "zone": "us-west-2a",
    "canary": false
  }
}
```

## 8. Listener API

### 8.1 List Listeners

```http
GET /api/v1/configurations/{config_id}/listeners
```

### 8.2 Create Listener

```http
POST /api/v1/configurations/{config_id}/listeners
```

**Request:**
```json
{
  "name": "http_listener",
  "address": "0.0.0.0",
  "port": 8080,
  "protocol": "HTTP",
  "filter_chains": [
    {
      "filters": [
        {
          "name": "envoy.filters.network.http_connection_manager",
          "config": {
            "stat_prefix": "http",
            "route_config": {
              "name": "local_route",
              "virtual_hosts": [...]
            },
            "http_filters": [
              {
                "name": "envoy.filters.http.router"
              }
            ]
          }
        }
      ]
    }
  ]
}
```

## 9. Route API

### 9.1 List Routes

```http
GET /api/v1/configurations/{config_id}/routes
```

### 9.2 Create Route

```http
POST /api/v1/configurations/{config_id}/routes
```

**Request:**
```json
{
  "name": "api_route",
  "virtual_host": "api.example.com",
  "domains": ["api.example.com", "api.example.org"],
  "routes": [
    {
      "match": {
        "prefix": "/v1/"
      },
      "route": {
        "cluster": "api_v1_cluster",
        "timeout_ms": 30000
      }
    },
    {
      "match": {
        "path": "/health"
      },
      "direct_response": {
        "status": 200,
        "body": {
          "inline_string": "OK"
        }
      }
    }
  ],
  "retry_policy": {
    "retry_on": "5xx",
    "num_retries": 3,
    "per_try_timeout_ms": 10000
  }
}
```

## 10. Certificate API

### 10.1 List Certificates

```http
GET /api/v1/certificates
```

### 10.2 Upload Certificate

```http
POST /api/v1/certificates
```

**Request:**
```json
{
  "name": "api-example-com",
  "certificate_pem": "-----BEGIN CERTIFICATE-----\n...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...",
  "certificate_chain": "-----BEGIN CERTIFICATE-----\n..."
}
```

### 10.3 Get Certificate

```http
GET /api/v1/certificates/{id}
```

**Response:**
```json
{
  "id": "cc0e8400-e29b-41d4-a716-446655440000",
  "name": "api-example-com",
  "subject": "CN=api.example.com",
  "issuer": "CN=Let's Encrypt Authority X3",
  "not_before": "2024-01-01T00:00:00Z",
  "not_after": "2024-04-01T00:00:00Z",
  "days_until_expiry": 75,
  "in_use_by": [
    {
      "type": "listener",
      "id": "dd0e8400-e29b-41d4-a716-446655440000",
      "name": "https_listener"
    }
  ]
}
```

## 11. Deployment API

### 11.1 List Deployments

```http
GET /api/v1/deployments
```

### 11.2 Create Deployment

```http
POST /api/v1/deployments
```

**Request:**
```json
{
  "proxy_ids": [
    "770e8400-e29b-41d4-a716-446655440000",
    "771e8400-e29b-41d4-a716-446655440001"
  ],
  "version_id": "bb0e8400-e29b-41d4-a716-446655440000",
  "deployment_strategy": "canary",
  "canary_config": {
    "percentage": 10,
    "duration_minutes": 30,
    "auto_promote": true
  }
}
```

### 11.3 Get Deployment Status

```http
GET /api/v1/deployments/{id}/status
```

**Response:**
```json
{
  "id": "ee0e8400-e29b-41d4-a716-446655440000",
  "status": "in_progress",
  "strategy": "canary",
  "started_at": "2024-01-15T10:00:00Z",
  "progress": {
    "total_proxies": 10,
    "completed": 3,
    "in_progress": 2,
    "pending": 5,
    "failed": 0
  },
  "canary_status": {
    "percentage_deployed": 30,
    "error_rate": 0.01,
    "latency_p99": 45.2,
    "auto_promote_at": "2024-01-15T10:30:00Z"
  }
}
```

## 12. Metrics API

### 12.1 Get Proxy Metrics

```http
GET /api/v1/proxies/{id}/metrics
```

**Query Parameters:**
- `start_time`: ISO 8601 timestamp
- `end_time`: ISO 8601 timestamp
- `resolution`: Time resolution (1m, 5m, 1h, 1d)
- `metrics`: Comma-separated list of metrics

**Response:**
```json
{
  "proxy_id": "770e8400-e29b-41d4-a716-446655440000",
  "time_range": {
    "start": "2024-01-15T09:00:00Z",
    "end": "2024-01-15T10:00:00Z"
  },
  "resolution": "5m",
  "metrics": {
    "requests_per_second": [
      {"timestamp": "2024-01-15T09:00:00Z", "value": 1250.5},
      {"timestamp": "2024-01-15T09:05:00Z", "value": 1275.3}
    ],
    "error_rate": [
      {"timestamp": "2024-01-15T09:00:00Z", "value": 0.01},
      {"timestamp": "2024-01-15T09:05:00Z", "value": 0.012}
    ],
    "p99_latency_ms": [
      {"timestamp": "2024-01-15T09:00:00Z", "value": 45.2},
      {"timestamp": "2024-01-15T09:05:00Z", "value": 47.8}
    ]
  }
}
```

### 12.2 Get Cluster Metrics

```http
GET /api/v1/clusters/{id}/metrics
```

### 12.3 Get System Metrics

```http
GET /api/v1/metrics/system
```

## 13. Audit Log API

### 13.1 List Audit Logs

```http
GET /api/v1/audit-logs
```

**Query Parameters:**
- `start_date`: Filter by start date
- `end_date`: Filter by end date
- `user_id`: Filter by user
- `action`: Filter by action type
- `resource_type`: Filter by resource type

**Response:**
```json
{
  "data": [
    {
      "id": "ff0e8400-e29b-41d4-a716-446655440000",
      "user": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "email": "user@example.com"
      },
      "action": "UPDATE",
      "resource_type": "proxy_configuration",
      "resource_id": "aa0e8400-e29b-41d4-a716-446655440000",
      "changes": {
        "before": {...},
        "after": {...}
      },
      "ip_address": "203.0.113.42",
      "user_agent": "EnvoyCtl/1.0",
      "timestamp": "2024-01-15T10:15:00Z"
    }
  ]
}
```

## 14. Webhook API

### 14.1 List Webhooks

```http
GET /api/v1/webhooks
```

### 14.2 Create Webhook

```http
POST /api/v1/webhooks
```

**Request:**
```json
{
  "name": "deployment-notifier",
  "url": "https://hooks.example.com/envoy",
  "events": [
    "deployment.started",
    "deployment.completed",
    "deployment.failed"
  ],
  "secret": "webhook_secret_key",
  "active": true
}
```

### 14.3 Test Webhook

```http
POST /api/v1/webhooks/{id}/test
```

## 15. Bulk Operations API

### 15.1 Bulk Update Proxies

```http
POST /api/v1/bulk/proxies/update
```

**Request:**
```json
{
  "proxy_ids": [
    "770e8400-e29b-41d4-a716-446655440000",
    "771e8400-e29b-41d4-a716-446655440001"
  ],
  "updates": {
    "cluster_id": "990e8400-e29b-41d4-a716-446655440000"
  }
}
```

### 15.2 Bulk Delete

```http
POST /api/v1/bulk/delete
```

**Request:**
```json
{
  "resource_type": "containers",
  "ids": [
    "880e8400-e29b-41d4-a716-446655440000",
    "881e8400-e29b-41d4-a716-446655440001"
  ]
}
```

## 16. WebSocket API

### 16.1 Real-time Updates

```javascript
ws://api.envoy-control.example.com/api/v1/ws

// Subscribe to events
{
  "type": "subscribe",
  "events": [
    "proxy.status.changed",
    "configuration.updated",
    "deployment.progress"
  ],
  "filters": {
    "proxy_ids": ["770e8400-e29b-41d4-a716-446655440000"]
  }
}

// Receive events
{
  "type": "event",
  "event": "proxy.status.changed",
  "data": {
    "proxy_id": "770e8400-e29b-41d4-a716-446655440000",
    "old_status": "healthy",
    "new_status": "unhealthy",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

## 17. API Rate Limiting

### 17.1 Rate Limit Headers

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1705313400
```

### 17.2 Rate Limit Rules

| Endpoint Pattern | Limit | Window |
|-----------------|-------|---------|
| /api/v1/auth/* | 5 | 1 minute |
| /api/v1/*/create | 100 | 1 minute |
| /api/v1/*/update | 100 | 1 minute |
| /api/v1/*/delete | 50 | 1 minute |
| /api/v1/*/list | 1000 | 1 minute |
| /api/v1/bulk/* | 10 | 1 minute |

## 18. API Versioning

### 18.1 Version Header

```http
API-Version: 1.0.0
```

### 18.2 Deprecation Notice

```http
Deprecation: true
Sunset: 2024-12-31T23:59:59Z
Link: <https://docs.example.com/migration>; rel="deprecation"
```

## 19. Pagination

### 19.1 Cursor-based Pagination

```http
GET /api/v1/proxies?cursor=eyJpZCI6IjEwMCJ9&limit=20
```

**Response:**
```json
{
  "data": [...],
  "cursor": {
    "next": "eyJpZCI6IjEyMCJ9",
    "previous": "eyJpZCI6IjgwIn0",
    "has_more": true
  }
}
```

## 20. Request Flow Diagrams

### 20.1 Configuration Update Flow

```mermaid
sequenceDiagram
    participant Client
    participant API as Axum API
    participant Validator
    participant Service
    participant SQLx
    participant DB as PostgreSQL
    participant Queue as NATS
    participant xDS
    participant Envoy

    Client->>API: PUT /proxies/{id}/configuration
    API->>Validator: Validate Configuration
    Validator-->>API: Validation Result

    alt Valid Configuration
        API->>Service: Update Configuration
        Service->>SQLx: BEGIN Transaction

        Service->>SQLx: Insert Configuration Version
        SQLx->>DB: INSERT INTO configuration_versions

        Service->>SQLx: Update Active Version
        SQLx->>DB: UPDATE proxy_configurations

        Service->>SQLx: COMMIT Transaction
        SQLx-->>Service: Success

        Service->>Queue: Publish Update Event
        Queue->>xDS: Configuration Updated
        xDS->>Envoy: Push New Config (xDS)
        Envoy-->>xDS: ACK

        Service-->>API: Update Success
        API-->>Client: 200 OK
    else Invalid Configuration
        Validator-->>API: Validation Errors
        API-->>Client: 422 Unprocessable Entity
    end
```

### 20.2 Authentication Flow

```mermaid
sequenceDiagram
    participant Client
    participant API as Axum API
    participant Auth as Auth Middleware
    participant JWT as JWT Service
    participant SQLx
    participant DB as PostgreSQL

    Client->>API: POST /auth/login
    API->>SQLx: Query User
    SQLx->>DB: SELECT FROM users
    DB-->>SQLx: User Record

    API->>API: Verify Password Hash

    alt Valid Credentials
        API->>JWT: Generate Tokens
        JWT-->>API: Access + Refresh Tokens
        API-->>Client: 200 OK + Tokens

        Client->>API: GET /proxies (with token)
        API->>Auth: Validate Token
        Auth->>JWT: Verify Token
        JWT-->>Auth: Claims
        Auth->>SQLx: Check User Status
        SQLx->>DB: SELECT FROM users
        DB-->>SQLx: User Active
        Auth-->>API: Authorized
        API-->>Client: 200 OK + Data
    else Invalid Credentials
        API-->>Client: 401 Unauthorized
    end
```

### 20.3 Deployment Flow

```mermaid
sequenceDiagram
    participant Client
    participant API as Axum API
    participant Deploy as Deployment Service
    participant SQLx
    participant DB as PostgreSQL
    participant Queue as NATS
    participant Agent as Node Agent
    participant Docker

    Client->>API: POST /deployments
    API->>Deploy: Create Deployment
    Deploy->>SQLx: Insert Deployment
    SQLx->>DB: INSERT INTO deployments

    Deploy->>Queue: Publish Deploy Command
    Queue->>Agent: Deploy Configuration

    loop For Each Container
        Agent->>Docker: Update Container
        Docker-->>Agent: Success/Failure
        Agent->>Queue: Report Status
        Queue->>Deploy: Update Status
        Deploy->>SQLx: Update Deployment
        SQLx->>DB: UPDATE deployments
    end

    Deploy-->>API: Deployment Status
    API-->>Client: 202 Accepted

    Client->>API: GET /deployments/{id}/status
    API->>SQLx: Query Status
    SQLx->>DB: SELECT FROM deployments
    DB-->>SQLx: Deployment Record
    API-->>Client: 200 OK + Status
```

## 21. OpenAPI Specification

The complete OpenAPI 3.0 specification is available at:
```
GET /api/v1/openapi.json
```

## 22. SDK Examples

### 22.1 Rust Client Example

```rust
use envoy_control_client::{Client, Configuration};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Initialize client
    let client = Client::new("https://api.envoy-control.example.com")
        .with_token("your_jwt_token");

    // List proxies
    let proxies = client.proxies().list().await?;

    // Update configuration
    let config = Configuration {
        static_resources: StaticResources {
            clusters: vec![...],
        },
        ..Default::default()
    };

    let result = client
        .proxies()
        .update_configuration("proxy_id", &config)
        .await?;

    Ok(())
}
```

### 22.2 CLI Usage

```bash
# Login
envoyctl auth login --email user@example.com

# List proxies
envoyctl proxy list --status healthy

# Update configuration
envoyctl proxy update-config proxy-001 --file config.yaml

# Deploy configuration
envoyctl deploy create --proxy-ids proxy-001,proxy-002 --version 5

# Watch deployment
envoyctl deploy watch deployment-123
```

## 23. Testing the API

### 23.1 Health Check

```bash
curl https://api.envoy-control.example.com/api/v1/health
```

### 23.2 Example CRUD Operations

```bash
# Create a VM
curl -X POST https://api.envoy-control.example.com/api/v1/virtual-machines \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "hostname": "vm-test-001",
    "ip_address": "10.0.1.10",
    "region": "us-west-2"
  }'

# List proxies with pagination
curl "https://api.envoy-control.example.com/api/v1/proxies?page=1&per_page=10" \
  -H "Authorization: Bearer $TOKEN"

# Update proxy configuration
curl -X PUT https://api.envoy-control.example.com/api/v1/proxies/770e8400/configuration \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @configuration.json
```

## 24. API Security Best Practices

1. **Always use HTTPS** in production
2. **Implement rate limiting** to prevent abuse
3. **Use strong JWT secrets** and rotate them regularly
4. **Validate all inputs** at API and database levels
5. **Implement CORS** properly for browser-based clients
6. **Use API versioning** for backward compatibility
7. **Log all API access** for audit purposes
8. **Implement request signing** for critical operations
9. **Use database transactions** for atomic operations
10. **Implement proper error handling** without exposing internals