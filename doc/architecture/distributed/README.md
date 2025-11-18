# Distributed Elsa Workflows Architecture

## Overview

This document describes the distributed architecture for deploying Elsa Workflows 3.0 in a high-availability, scalable environment. The architecture is designed to support enterprise-grade workflow execution with horizontal scalability, fault tolerance, and reliable message delivery.

## Architecture Goals

- **Horizontal Scalability**: Support scaling worker nodes independently based on workload
- **High Availability**: Eliminate single points of failure through redundancy
- **Reliable Execution**: Ensure workflows complete even in the face of node failures
- **Performance**: Optimize for high-throughput workflow execution
- **Observability**: Comprehensive monitoring and logging capabilities

## Key Technologies

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Workflow Engine | Elsa Workflows 3.0 (.NET 9.0) | Core workflow execution engine |
| Persistence | PostgreSQL 15+ | Workflow definitions and state storage |
| Message Queue | AWS SQS | Task distribution and decoupling |
| Cache | Redis | Performance optimization |
| API | ASP.NET Core REST API | Workflow management interface |
| UI | Blazor WebAssembly | Visual workflow designer |

## Architecture Components

### 1. API Gateway
- Entry point for all client requests
- Handles authentication and authorization
- Routes requests to appropriate backend services
- Implements rate limiting and throttling

### 2. Workflow Server
- Hosts the Elsa workflow runtime
- Manages workflow lifecycle (create, start, pause, resume, cancel)
- Handles synchronous workflow execution
- Publishes long-running tasks to SQS

### 3. Workflow Runtime Engine
- Core Elsa.Workflows.Runtime component
- Executes workflow activities
- Manages bookmarks and state transitions
- Handles activity execution pipelines

### 4. Worker Nodes
- Horizontally scalable task processors
- Poll SQS for workflow tasks
- Execute activities and update state
- Can be scaled up/down based on queue depth

### 5. Workflow Scheduler
- Manages time-based triggers (cron, timers)
- Uses distributed locking to prevent duplicate execution
- Enqueues scheduled workflows to SQS

### 6. PostgreSQL Databases

#### Workflow Database (Main)
- Workflow definitions and versions
- Workflow instances and execution history
- Activity execution logs
- Bookmarks and triggers
- User and tenant data

#### Runtime State Store
- Distributed locks for coordination
- Execution state snapshots
- Transient runtime data
- Worker health and registration

### 7. AWS SQS Message Queue
- Distributes workflow tasks across workers
- Provides retry mechanisms with DLQ
- Ensures at-least-once delivery
- Decouples producers from consumers

### 8. Redis Distributed Cache
- Caches workflow definitions for fast access
- Stores activity type metadata
- Session state (if needed)
- Reduces database load

### 9. Workflow Studio
- Web-based visual designer
- Blazor WebAssembly application
- Communicates with API Gateway
- Real-time workflow monitoring

## Data Flow

### Workflow Creation
1. User designs workflow in Studio
2. Studio sends definition to API Gateway
3. API Gateway validates and stores in PostgreSQL
4. Definition is cached in Redis

### Synchronous Workflow Execution
1. External trigger calls API Gateway
2. API Gateway routes to Workflow Server
3. Workflow Server executes via Runtime Engine
4. State persisted to PostgreSQL
5. Result returned to caller

### Asynchronous Workflow Execution
1. Workflow Server receives execution request
2. Creates workflow instance in PostgreSQL
3. Publishes task message to SQS
4. Returns acknowledgment to caller
5. Worker node polls SQS and receives task
6. Worker executes workflow via Runtime Engine
7. Worker updates state in PostgreSQL
8. Worker acknowledges message to SQS

### Scheduled Workflow Execution
1. Scheduler periodically checks for due workflows
2. Acquires distributed lock in PostgreSQL
3. Enqueues workflow task to SQS
4. Releases lock
5. Worker processes as asynchronous execution

## Scalability Considerations

### Horizontal Scaling
- **Worker Nodes**: Scale based on SQS queue depth and message age
- **API Gateway**: Load balance across multiple instances
- **Workflow Server**: Run multiple instances behind load balancer

### Vertical Scaling
- **PostgreSQL**: Use read replicas for query load distribution
- **Redis**: Use Redis Cluster for large cache sizes

### Auto-Scaling Triggers
- SQS queue depth > 100 messages
- Average message age > 5 minutes
- CPU utilization > 70%
- Memory utilization > 80%

## High Availability

### Redundancy
- Multiple worker nodes across availability zones
- PostgreSQL with streaming replication
- Redis Sentinel for cache availability
- SQS inherently highly available

### Failure Handling
- Worker node failure: SQS redelivers message to healthy node
- Database failure: Automatic failover to standby
- Cache failure: Graceful degradation, read from database
- API Gateway failure: Load balancer routes to healthy instance

### Distributed Locking
- PostgreSQL advisory locks for scheduler coordination
- Prevents duplicate scheduled execution
- Timeout-based lock release for crash recovery

## Monitoring and Observability

### Metrics
- Workflow execution duration
- Activity execution counts
- Queue depth and processing rate
- Database connection pool usage
- Cache hit/miss rates
- Worker node health

### Logging
- Structured logging with correlation IDs
- Workflow execution traces
- Error and exception tracking
- Audit logs for compliance

### Tracing
- Distributed tracing with OpenTelemetry
- Request flow across components
- Performance bottleneck identification

### Recommended Tools
- Application Insights (Azure)
- CloudWatch (AWS)
- Datadog
- Grafana + Prometheus

## Security Considerations

### Authentication & Authorization
- OAuth 2.0 / OpenID Connect
- JWT tokens for API access
- Role-based access control (RBAC)
- Tenant isolation

### Network Security
- VPC isolation
- Private subnets for databases
- Security groups / firewall rules
- TLS/SSL for all communications

### Data Security
- Encryption at rest (PostgreSQL)
- Encryption in transit (TLS 1.3)
- Sensitive data masking in logs
- Regular security updates

## Deployment Topology

### Recommended AWS Setup
```
Region: us-east-1
├── VPC (10.0.0.0/16)
│   ├── Public Subnet AZ-A (10.0.1.0/24)
│   │   └── NAT Gateway
│   ├── Public Subnet AZ-B (10.0.2.0/24)
│   │   └── NAT Gateway
│   ├── Private Subnet AZ-A (10.0.11.0/24)
│   │   ├── Workflow Server (ASG)
│   │   └── Worker Nodes (ASG)
│   ├── Private Subnet AZ-B (10.0.12.0/24)
│   │   ├── Workflow Server (ASG)
│   │   └── Worker Nodes (ASG)
│   ├── Data Subnet AZ-A (10.0.21.0/24)
│   │   ├── PostgreSQL Primary
│   │   └── Redis Primary
│   └── Data Subnet AZ-B (10.0.22.0/24)
│       ├── PostgreSQL Standby
│       └── Redis Replica
├── Application Load Balancer
├── SQS Queues
│   ├── workflow-tasks.fifo
│   └── workflow-tasks-dlq.fifo
└── S3 (configuration, backups)
```

## Performance Optimization

### Database Optimization
- Connection pooling (min: 10, max: 100)
- Prepared statements for frequent queries
- Indexes on foreign keys and query predicates
- Partitioning for large tables (execution logs)

### Caching Strategy
- Cache workflow definitions (TTL: 1 hour)
- Cache activity type metadata (TTL: 24 hours)
- Invalidate on workflow update
- Use Redis cluster for high throughput

### Message Queue Tuning
- Batch size: 10 messages per poll
- Visibility timeout: 5 minutes
- Long polling: 20 seconds
- DLQ max receives: 3

### Worker Node Optimization
- Async/await throughout
- Parallel activity execution where possible
- Connection pooling
- Graceful shutdown handling

## Cost Optimization

### Compute
- Use spot instances for worker nodes
- Auto-scaling based on demand
- Right-size instance types

### Storage
- Use S3 for large workflow artifacts
- Archive old execution logs to S3 Glacier
- Optimize PostgreSQL storage tier

### Data Transfer
- Keep components in same region/AZ
- Use VPC endpoints for AWS services
- Compress large payloads

## Disaster Recovery

### Backup Strategy
- PostgreSQL: Daily full backups, hourly incrementals
- Backup retention: 30 days
- Cross-region backup replication
- Regular restore testing

### RTO/RPO Targets
- Recovery Time Objective (RTO): 1 hour
- Recovery Point Objective (RPO): 15 minutes

### Recovery Procedures
1. Promote standby database to primary
2. Update DNS/load balancer to new region
3. Scale up worker nodes in DR region
4. Verify workflow execution
5. Monitor for issues

## Migration Path

### From Single-Node to Distributed

1. **Phase 1: Add Message Queue**
   - Introduce SQS alongside existing server
   - Configure workflow server to publish to SQS
   - Test with subset of workflows

2. **Phase 2: Add Worker Nodes**
   - Deploy initial worker nodes
   - Configure to consume from SQS
   - Gradually increase worker count

3. **Phase 3: Add Caching**
   - Deploy Redis cluster
   - Configure workflow server to use cache
   - Monitor cache hit rates

4. **Phase 4: High Availability**
   - Set up PostgreSQL replication
   - Deploy multiple workflow servers
   - Configure load balancer

5. **Phase 5: Full Production**
   - Enable auto-scaling
   - Configure monitoring and alerts
   - Implement disaster recovery

## Further Reading

- [C4 Container Diagram](diagrams/c4-container-diagram.puml)
- [Component Details](components/)
- [Deployment Guide](deployment-guide.md)
- [Operations Runbook](operations-runbook.md)

## References

- [Elsa Workflows Documentation](https://v3.elsaworkflows.io/)
- [AWS SQS Best Practices](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-best-practices.html)
- [PostgreSQL High Availability](https://www.postgresql.org/docs/current/high-availability.html)
- [Redis Cluster Tutorial](https://redis.io/docs/manual/scaling/)
