# Worker Nodes Component

## Overview
Worker nodes are horizontally scalable compute instances that process workflow tasks from the SQS queue. They are designed for high throughput, fault tolerance, and elastic scaling based on workload demand.

## Technology Stack
- **Framework**: .NET 9.0
- **Host**: Generic Host (Microsoft.Extensions.Hosting)
- **Queue Client**: AWS SDK for .NET (AWSSDK.SQS)
- **Workflow Engine**: Elsa.Workflows.Runtime

## Architecture

### Worker Node Structure
```
Worker Node
├── Queue Poller Service (Background Service)
├── Task Executor (Workflow Runtime)
├── State Persistence Layer (PostgreSQL)
├── Health Monitor
└── Graceful Shutdown Handler
```

## Responsibilities

### Queue Processing
- Long-poll SQS for workflow tasks
- Batch message retrieval (up to 10 messages)
- Message visibility timeout management
- Automatic message acknowledgment/rejection

### Workflow Execution
- Deserialize workflow task from message
- Execute workflow activities via Runtime Engine
- Handle activity failures and retries
- Persist execution state to database

### State Management
- Update workflow instance status
- Record activity execution logs
- Manage bookmarks and continuations
- Handle distributed transactions

### Health Monitoring
- Report node health to control plane
- Graceful degradation on resource constraints
- Self-healing for transient failures

## Message Format

### Task Message (JSON)
```json
{
  "messageId": "msg-123456",
  "taskType": "ExecuteWorkflow",
  "workflowInstanceId": "wf-instance-789",
  "bookmarkId": "bookmark-456",
  "correlationId": "corr-abc-def",
  "payload": {
    "input": {
      "orderId": "12345",
      "customerId": "cust-999"
    }
  },
  "metadata": {
    "tenantId": "tenant-001",
    "priority": "normal",
    "maxRetries": 3,
    "timeout": 300
  },
  "enqueuedAt": "2025-11-18T10:30:00Z"
}
```

## Processing Flow

### 1. Message Polling
```csharp
while (!cancellationToken.IsCancellationRequested)
{
    var messages = await sqsClient.ReceiveMessageAsync(new ReceiveMessageRequest
    {
        QueueUrl = queueUrl,
        MaxNumberOfMessages = 10,
        WaitTimeSeconds = 20, // Long polling
        VisibilityTimeout = 300, // 5 minutes
        MessageAttributeNames = new List<string> { "All" }
    }, cancellationToken);

    foreach (var message in messages.Messages)
    {
        await ProcessMessageAsync(message, cancellationToken);
    }
}
```

### 2. Message Processing
```csharp
async Task ProcessMessageAsync(Message message, CancellationToken ct)
{
    try
    {
        // Deserialize task
        var task = JsonSerializer.Deserialize<WorkflowTask>(message.Body);
        
        // Execute workflow
        var result = await workflowRuntime.ExecuteWorkflowAsync(
            task.WorkflowInstanceId,
            task.BookmarkId,
            task.Payload,
            ct);
        
        // Persist state
        await stateStore.SaveExecutionResultAsync(result, ct);
        
        // Delete message from queue
        await sqsClient.DeleteMessageAsync(queueUrl, message.ReceiptHandle, ct);
        
        logger.LogInformation("Successfully processed task {TaskId}", task.MessageId);
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to process task {MessageId}", message.MessageId);
        // Message will return to queue after visibility timeout
    }
}
```

### 3. Execution Result Handling
- **Success**: Delete message, persist final state
- **Activity Failure**: Retry activity based on policy
- **Workflow Suspension**: Persist bookmark, delete message
- **Fatal Error**: Move to DLQ, log error, alert

## Configuration

### appsettings.json
```json
{
  "WorkerNode": {
    "NodeId": "worker-001",
    "MaxConcurrency": 10,
    "PollingInterval": "00:00:20",
    "VisibilityTimeout": "00:05:00",
    "BatchSize": 10,
    "ShutdownTimeout": "00:01:00"
  },
  "SQS": {
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456/workflow-tasks.fifo",
    "Region": "us-east-1",
    "MaxRetries": 3,
    "RequestTimeout": "00:00:30"
  },
  "Database": {
    "ConnectionString": "Host=postgres-cluster;Database=elsa_workflows;Username=elsa_worker;Password=***",
    "MaxPoolSize": 50,
    "MinPoolSize": 5,
    "CommandTimeout": 300
  },
  "Redis": {
    "Configuration": "redis-cluster:6379",
    "InstanceName": "ElsaWorkflows:",
    "ConnectTimeout": 5000
  }
}
```

## Scaling Strategy

### Horizontal Scaling
```yaml
# Kubernetes HPA Example
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: elsa-worker-nodes
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: elsa-worker-nodes
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: sqs_queue_depth
      target:
        type: AverageValue
        averageValue: "10"
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Scaling Triggers
- **Scale Up**: Queue depth > 100 OR message age > 5 minutes
- **Scale Down**: Queue depth < 10 AND CPU < 30% for 10 minutes
- **Cooldown**: 5 minutes between scale operations

## High Availability

### Fault Tolerance
- Multiple worker nodes across availability zones
- Automatic task redistribution on node failure
- SQS message redelivery on processing failure
- Database connection retry with exponential backoff

### Graceful Shutdown
```csharp
public override async Task StopAsync(CancellationToken cancellationToken)
{
    logger.LogInformation("Worker node shutdown initiated");
    
    // Stop accepting new messages
    stopPolling = true;
    
    // Wait for in-flight tasks to complete
    await Task.WhenAll(activeTasks);
    
    // Flush pending database writes
    await stateStore.FlushAsync();
    
    // Close connections
    await CloseConnectionsAsync();
    
    logger.LogInformation("Worker node shutdown complete");
}
```

### Health Checks
- **Liveness**: Can the process respond?
- **Readiness**: Can it accept new work?
- **Startup**: Is initialization complete?

```csharp
services.AddHealthChecks()
    .AddCheck("DatabaseConnection", () => CheckDatabase())
    .AddCheck("QueueConnection", () => CheckSQS())
    .AddCheck("WorkflowRuntime", () => CheckRuntime())
    .AddCheck("MemoryUsage", () => CheckMemory());
```

## Error Handling

### Retry Strategy
1. **Immediate Retry**: Activity-level errors (3 attempts)
2. **Visibility Timeout**: Message returns to queue
3. **Exponential Backoff**: Increasing delays between retries
4. **Dead Letter Queue**: After max retries exceeded

### Error Classification
- **Transient**: Network errors, temporary DB unavailability → Retry
- **Permanent**: Validation errors, missing resources → DLQ
- **Timeout**: Long-running activities → Cancel and reschedule

### DLQ Processing
- Separate consumer for DLQ analysis
- Alert on DLQ depth > 10
- Manual intervention workflow
- Automated reprocessing for known issues

## Performance Optimization

### Concurrency Control
```csharp
var semaphore = new SemaphoreSlim(maxConcurrency);

foreach (var message in messages)
{
    await semaphore.WaitAsync();
    _ = Task.Run(async () =>
    {
        try
        {
            await ProcessMessageAsync(message);
        }
        finally
        {
            semaphore.Release();
        }
    });
}
```

### Connection Pooling
- Database: 50 connections per node
- Redis: 10 multiplexer connections
- HTTP clients: Shared instances

### Batch Processing
- Retrieve up to 10 messages per SQS call
- Batch database writes where possible
- Aggregate logs before writing

### Memory Management
- Limit workflow instance cache size
- Stream large payloads
- Dispose resources promptly
- Monitor GC pressure

## Monitoring

### Key Metrics
- **Throughput**: Tasks processed per minute
- **Latency**: Time from queue to completion
- **Queue Depth**: Messages waiting in SQS
- **Error Rate**: Failed tasks / total tasks
- **Resource Usage**: CPU, memory, network

### Logging
```json
{
  "timestamp": "2025-11-18T10:35:00Z",
  "level": "Information",
  "message": "Workflow task completed",
  "nodeId": "worker-001",
  "taskId": "msg-123456",
  "workflowInstanceId": "wf-instance-789",
  "duration": 2345,
  "activitiesExecuted": 12,
  "status": "Completed"
}
```

### Alerts
- Task processing time > 5 minutes
- Error rate > 5% over 5 minutes
- Queue depth > 500 for 10 minutes
- DLQ depth > 10
- Node memory usage > 90%

## Security

### IAM Permissions (AWS)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456:workflow-tasks.fifo"
    }
  ]
}
```

### Network Security
- Private subnet deployment
- No direct internet access
- VPC endpoints for AWS services
- Encrypted connections to database/cache

### Secrets Management
- AWS Secrets Manager or Azure Key Vault
- Rotate credentials regularly
- Environment variable injection
- No secrets in logs

## Deployment

### Docker Image
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Elsa.WorkerNode.dll"]
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elsa-worker-nodes
spec:
  replicas: 5
  selector:
    matchLabels:
      app: elsa-worker
  template:
    metadata:
      labels:
        app: elsa-worker
    spec:
      containers:
      - name: worker
        image: elsa-worker:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        env:
        - name: WorkerNode__NodeId
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

## Troubleshooting

### Common Issues

**High message age in queue**
- Scale up worker nodes
- Check for worker node failures
- Investigate slow database queries

**DLQ accumulation**
- Review error logs for patterns
- Check for data validation issues
- Verify external service availability

**Memory leaks**
- Enable GC logging
- Capture heap dumps
- Review disposal patterns

**Database connection exhaustion**
- Increase connection pool size
- Reduce query timeout
- Optimize slow queries

## Related Components
- [Workflow Runtime](workflow-runtime.md)
- [Message Queue (SQS)](message-queue.md)
- [Database Schema](database-schema.md)
