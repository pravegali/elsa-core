# API Gateway Component

## Overview
The API Gateway serves as the single entry point for all client interactions with the Elsa Workflows platform. It handles request routing, authentication, authorization, and cross-cutting concerns.

## Technology Stack
- **Framework**: ASP.NET Core 9.0
- **API Style**: RESTful JSON API
- **Authentication**: JWT Bearer Tokens (OAuth 2.0)
- **Documentation**: OpenAPI/Swagger

## Responsibilities

### Request Routing
- Routes workflow management requests to Workflow Server
- Handles CRUD operations on workflow definitions
- Manages workflow instance lifecycle operations

### Authentication & Authorization
- Validates JWT tokens
- Integrates with identity providers (Azure AD, Auth0, Keycloak)
- Enforces role-based access control (RBAC)
- Supports tenant-based isolation

### Rate Limiting
- Per-user rate limiting
- Per-tenant quotas
- DDoS protection
- Backpressure management

### Cross-Cutting Concerns
- Request/response logging
- Correlation ID propagation
- Error handling and standardized responses
- API versioning

## API Endpoints

### Workflow Definitions
```
GET    /api/workflows/definitions          # List all definitions
GET    /api/workflows/definitions/{id}     # Get specific definition
POST   /api/workflows/definitions          # Create new definition
PUT    /api/workflows/definitions/{id}     # Update definition
DELETE /api/workflows/definitions/{id}     # Delete definition
POST   /api/workflows/definitions/{id}/publish  # Publish version
```

### Workflow Instances
```
GET    /api/workflows/instances            # List instances
GET    /api/workflows/instances/{id}       # Get instance details
POST   /api/workflows/instances/execute    # Execute workflow
POST   /api/workflows/instances/{id}/cancel    # Cancel execution
POST   /api/workflows/instances/{id}/resume    # Resume from bookmark
GET    /api/workflows/instances/{id}/journal   # Get execution log
```

### Activity Definitions
```
GET    /api/activities/types               # List available activities
GET    /api/activities/types/{type}        # Get activity metadata
```

### System Management
```
GET    /api/system/health                  # Health check
GET    /api/system/metrics                 # Performance metrics
GET    /api/system/version                 # API version info
```

## Configuration

### appsettings.json
```json
{
  "ApiGateway": {
    "RateLimiting": {
      "EnableRateLimiting": true,
      "PermitLimit": 100,
      "Window": "00:01:00",
      "QueueLimit": 10
    },
    "Authentication": {
      "Authority": "https://login.microsoftonline.com/{tenant-id}",
      "Audience": "api://elsa-workflows",
      "ValidateIssuer": true,
      "RequireHttpsMetadata": true
    },
    "WorkflowServer": {
      "BaseUrl": "http://workflow-server:5000",
      "Timeout": "00:05:00"
    },
    "Cors": {
      "AllowedOrigins": ["https://studio.example.com"],
      "AllowCredentials": true
    }
  }
}
```

## High Availability Setup

### Load Balancing
- Deploy multiple API Gateway instances (minimum 2)
- Use Application Load Balancer (ALB) or Azure Load Balancer
- Health check endpoint: `/api/system/health`
- Session affinity: Not required (stateless)

### Scaling Strategy
- Auto-scale based on:
  - CPU utilization > 70%
  - Request count > 1000/min per instance
  - Response time > 500ms (p95)

### Deployment
- Rolling updates with zero downtime
- Blue-green deployment support
- Canary releases for testing

## Security

### Authentication Flow
1. Client requests token from Identity Provider
2. Client includes token in `Authorization` header
3. API Gateway validates token signature and claims
4. Extracts user/tenant context
5. Forwards authenticated request to backend

### Authorization Policies
- **Admin**: Full access to all operations
- **Developer**: Create/update workflows, view executions
- **Operator**: Execute workflows, view status
- **Viewer**: Read-only access

### Network Security
- TLS 1.3 for all external connections
- mTLS for internal service communication
- IP whitelisting for admin endpoints
- WAF (Web Application Firewall) integration

## Monitoring

### Metrics
- Request rate (requests/second)
- Response times (p50, p95, p99)
- Error rates by endpoint
- Authentication success/failure rates
- Rate limit rejections

### Alerts
- Error rate > 5% for 5 minutes
- Response time p95 > 1 second
- Authentication failures > 100/minute
- Health check failures

### Logging
```json
{
  "timestamp": "2025-11-18T10:30:00Z",
  "level": "Information",
  "message": "Workflow execution started",
  "correlationId": "abc-123-def-456",
  "userId": "user@example.com",
  "tenantId": "tenant-001",
  "endpoint": "/api/workflows/instances/execute",
  "method": "POST",
  "statusCode": 202,
  "duration": 125
}
```

## Error Handling

### Standard Error Response
```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "The workflow definition contains validation errors",
  "instance": "/api/workflows/definitions",
  "correlationId": "abc-123-def-456",
  "errors": {
    "name": ["The name field is required"],
    "activities": ["At least one activity is required"]
  }
}
```

### HTTP Status Codes
- `200 OK`: Successful request
- `201 Created`: Resource created
- `202 Accepted`: Async operation started
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Missing/invalid authentication
- `403 Forbidden`: Insufficient permissions
- `404 Not Found`: Resource not found
- `409 Conflict`: Resource conflict
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error
- `503 Service Unavailable`: Temporary unavailability

## Performance Optimization

### Caching
- Cache workflow definitions (5 minutes)
- Cache activity type metadata (1 hour)
- ETag support for conditional requests
- Cache-Control headers

### Connection Pooling
```csharp
services.AddHttpClient("WorkflowServer")
    .ConfigureHttpClient(client => {
        client.BaseAddress = new Uri(config.WorkflowServer.BaseUrl);
        client.Timeout = config.WorkflowServer.Timeout;
    })
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler {
        PooledConnectionLifetime = TimeSpan.FromMinutes(5),
        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
        MaxConnectionsPerServer = 100
    });
```

### Response Compression
- Gzip/Brotli compression enabled
- Minimum response size: 1KB
- Compressed content types: JSON, XML, text

## Disaster Recovery

### Circuit Breaker
- Open circuit after 5 consecutive failures
- Half-open state after 30 seconds
- Close circuit after 3 successful requests

### Graceful Degradation
- Return cached data when backend unavailable
- Queue write operations for retry
- Return partial results with warnings

### Health Checks
```csharp
services.AddHealthChecks()
    .AddCheck("WorkflowServer", () => CheckWorkflowServer())
    .AddCheck("Database", () => CheckDatabase())
    .AddCheck("Redis", () => CheckRedis())
    .AddCheck("SQS", () => CheckSQS());
```

## Development

### Local Setup
```bash
cd src/apps/Elsa.ApiGateway
dotnet restore
dotnet run
```

### Testing
```bash
# Unit tests
dotnet test test/unit/Elsa.ApiGateway.Tests

# Integration tests
dotnet test test/integration/Elsa.ApiGateway.IntegrationTests

# Load testing
k6 run scripts/load-tests/api-gateway.js
```

## Related Components
- [Workflow Server](workflow-server.md)
- [Workflow Studio](workflow-studio.md)
- [Authentication](authentication.md)
