# AWS SQS Message Queue Component

## Overview
Amazon Simple Queue Service (SQS) provides the message queuing infrastructure for distributing workflow tasks across worker nodes in the Elsa Workflows distributed architecture.

## Queue Architecture

### Queue Types

#### 1. Workflow Tasks Queue (FIFO)
**Name**: `workflow-tasks.fifo`

**Purpose**: Primary queue for workflow execution tasks

**Configuration**:
- **Type**: FIFO (First-In-First-Out)
- **Content-based deduplication**: Enabled
- **Message retention**: 4 days
- **Visibility timeout**: 5 minutes
- **Receive wait time**: 20 seconds (long polling)
- **Max message size**: 256 KB

#### 2. Dead Letter Queue (DLQ)
**Name**: `workflow-tasks-dlq.fifo`

**Purpose**: Captures failed messages after max retries

**Configuration**:
- **Type**: FIFO
- **Message retention**: 14 days
- **Redrive policy**: Source queue after 3 retries

## Message Structure

### Task Message Schema
```json
{
  "messageId": "msg-123456789",
  "messageGroupId": "workflow-def-abc",
  "messageDeduplicationId": "unique-task-id-123",
  "taskType": "ExecuteWorkflow",
  "workflowInstanceId": "wf-inst-789",
  "bookmarkId": "bookmark-456",
  "correlationId": "corr-abc-def",
  "payload": {
    "input": {
      "orderId": "12345",
      "customerId": "cust-999"
    },
    "variables": {
      "priority": "high"
    }
  },
  "metadata": {
    "tenantId": "tenant-001",
    "priority": "normal",
    "maxRetries": 3,
    "timeout": 300,
    "source": "api-gateway"
  },
  "enqueuedAt": "2025-11-18T10:30:00Z",
  "attemptCount": 0
}
```

### Message Attributes
```json
{
  "TenantId": {
    "DataType": "String",
    "StringValue": "tenant-001"
  },
  "Priority": {
    "DataType": "String",
    "StringValue": "high"
  },
  "WorkflowDefinitionId": {
    "DataType": "String",
    "StringValue": "workflow-def-abc"
  },
  "CorrelationId": {
    "DataType": "String",
    "StringValue": "corr-abc-def"
  }
}
```

## Queue Operations

### Publishing Messages

#### .NET SDK Example
```csharp
using Amazon.SQS;
using Amazon.SQS.Model;

public class WorkflowTaskPublisher
{
    private readonly IAmazonSQS _sqsClient;
    private readonly string _queueUrl;

    public async Task PublishTaskAsync(WorkflowTask task, CancellationToken ct)
    {
        var messageBody = JsonSerializer.Serialize(task);
        
        var request = new SendMessageRequest
        {
            QueueUrl = _queueUrl,
            MessageBody = messageBody,
            MessageGroupId = task.WorkflowDefinitionId, // FIFO grouping
            MessageDeduplicationId = task.MessageId, // Deduplication
            MessageAttributes = new Dictionary<string, MessageAttributeValue>
            {
                ["TenantId"] = new MessageAttributeValue 
                { 
                    DataType = "String", 
                    StringValue = task.Metadata.TenantId 
                },
                ["Priority"] = new MessageAttributeValue 
                { 
                    DataType = "String", 
                    StringValue = task.Metadata.Priority 
                }
            }
        };

        var response = await _sqsClient.SendMessageAsync(request, ct);
        
        _logger.LogInformation(
            "Published task {TaskId} to queue with MessageId {MessageId}",
            task.MessageId, response.MessageId);
    }
}
```

### Batch Publishing
```csharp
public async Task PublishBatchAsync(List<WorkflowTask> tasks, CancellationToken ct)
{
    var entries = tasks.Select((task, index) => new SendMessageBatchRequestEntry
    {
        Id = index.ToString(),
        MessageBody = JsonSerializer.Serialize(task),
        MessageGroupId = task.WorkflowDefinitionId,
        MessageDeduplicationId = task.MessageId,
        MessageAttributes = CreateMessageAttributes(task)
    }).ToList();

    // SQS supports up to 10 messages per batch
    foreach (var batch in entries.Chunk(10))
    {
        var request = new SendMessageBatchRequest
        {
            QueueUrl = _queueUrl,
            Entries = batch.ToList()
        };

        var response = await _sqsClient.SendMessageBatchAsync(request, ct);
        
        if (response.Failed.Any())
        {
            _logger.LogWarning(
                "Failed to publish {Count} messages", 
                response.Failed.Count);
        }
    }
}
```

### Consuming Messages

#### Long Polling Example
```csharp
public class WorkflowTaskConsumer : BackgroundService
{
    private readonly IAmazonSQS _sqsClient;
    private readonly string _queueUrl;
    private readonly int _maxMessages = 10;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                var request = new ReceiveMessageRequest
                {
                    QueueUrl = _queueUrl,
                    MaxNumberOfMessages = _maxMessages,
                    WaitTimeSeconds = 20, // Long polling
                    VisibilityTimeout = 300, // 5 minutes
                    MessageAttributeNames = new List<string> { "All" },
                    AttributeNames = new List<string> { "All" }
                };

                var response = await _sqsClient.ReceiveMessageAsync(request, ct);

                foreach (var message in response.Messages)
                {
                    await ProcessMessageAsync(message, ct);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error polling queue");
                await Task.Delay(TimeSpan.FromSeconds(5), ct);
            }
        }
    }

    private async Task ProcessMessageAsync(Message message, CancellationToken ct)
    {
        try
        {
            var task = JsonSerializer.Deserialize<WorkflowTask>(message.Body);
            
            // Process the workflow task
            await _workflowExecutor.ExecuteAsync(task, ct);
            
            // Delete message on success
            await _sqsClient.DeleteMessageAsync(new DeleteMessageRequest
            {
                QueueUrl = _queueUrl,
                ReceiptHandle = message.ReceiptHandle
            }, ct);
            
            _logger.LogInformation("Successfully processed message {MessageId}", message.MessageId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process message {MessageId}", message.MessageId);
            // Message will return to queue after visibility timeout
        }
    }
}
```

### Visibility Timeout Extension
```csharp
// For long-running tasks, extend visibility timeout
private async Task ExtendVisibilityTimeoutAsync(
    string receiptHandle, 
    int additionalSeconds,
    CancellationToken ct)
{
    await _sqsClient.ChangeMessageVisibilityAsync(new ChangeMessageVisibilityRequest
    {
        QueueUrl = _queueUrl,
        ReceiptHandle = receiptHandle,
        VisibilityTimeout = additionalSeconds
    }, ct);
}
```

## FIFO Queue Considerations

### Message Group ID
- Groups related messages for ordered processing
- Use workflow definition ID for grouping
- Enables parallel processing across different workflows
- Messages within same group processed sequentially

### Message Deduplication
- Content-based deduplication enabled
- Deduplication window: 5 minutes
- Use unique task ID for deduplication ID
- Prevents duplicate workflow executions

### Throughput
- **Standard limit**: 300 messages/second per message group
- **High throughput**: 3,000 messages/second (across all groups)
- Enable high throughput FIFO for better performance

## Error Handling

### Retry Strategy

#### Application-Level Retries
```csharp
public async Task<bool> ProcessWithRetryAsync(
    Message message, 
    int maxAttempts = 3,
    CancellationToken ct = default)
{
    for (int attempt = 1; attempt <= maxAttempts; attempt++)
    {
        try
        {
            await ProcessMessageAsync(message, ct);
            return true;
        }
        catch (TransientException ex)
        {
            _logger.LogWarning(ex, 
                "Transient error on attempt {Attempt}/{MaxAttempts}", 
                attempt, maxAttempts);
            
            if (attempt < maxAttempts)
            {
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)), ct);
            }
        }
        catch (PermanentException ex)
        {
            _logger.LogError(ex, "Permanent error, moving to DLQ");
            return false; // Don't retry
        }
    }
    
    return false;
}
```

#### SQS-Level Retries
- Message returns to queue after visibility timeout
- Max receive count: 3 attempts
- After max receives, message moves to DLQ

### Dead Letter Queue Processing
```csharp
public class DlqProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var response = await _sqsClient.ReceiveMessageAsync(new ReceiveMessageRequest
            {
                QueueUrl = _dlqUrl,
                MaxNumberOfMessages = 10,
                WaitTimeSeconds = 20
            }, ct);

            foreach (var message in response.Messages)
            {
                await AnalyzeFailedMessageAsync(message, ct);
                
                // Store in database for manual review
                await _failureStore.SaveAsync(new FailedTask
                {
                    MessageId = message.MessageId,
                    Body = message.Body,
                    Attributes = message.Attributes,
                    FailedAt = DateTime.UtcNow,
                    RequiresInvestigation = true
                });
                
                // Delete from DLQ
                await _sqsClient.DeleteMessageAsync(_dlqUrl, message.ReceiptHandle, ct);
            }
            
            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}
```

## Monitoring and Metrics

### CloudWatch Metrics
- **ApproximateNumberOfMessagesVisible**: Queue depth
- **ApproximateAgeOfOldestMessage**: Message age
- **NumberOfMessagesSent**: Publish rate
- **NumberOfMessagesReceived**: Consume rate
- **NumberOfMessagesDeleted**: Success rate
- **NumberOfEmptyReceives**: Polling efficiency

### Custom Metrics
```csharp
public class QueueMetricsPublisher
{
    private readonly IAmazonCloudWatch _cloudWatch;

    public async Task PublishMetricsAsync(CancellationToken ct)
    {
        var queueAttributes = await _sqsClient.GetQueueAttributesAsync(
            new GetQueueAttributesRequest
            {
                QueueUrl = _queueUrl,
                AttributeNames = new List<string> 
                { 
                    "ApproximateNumberOfMessages",
                    "ApproximateNumberOfMessagesNotVisible"
                }
            }, ct);

        await _cloudWatch.PutMetricDataAsync(new PutMetricDataRequest
        {
            Namespace = "ElsaWorkflows/Queue",
            MetricData = new List<MetricDatum>
            {
                new MetricDatum
                {
                    MetricName = "QueueDepth",
                    Value = double.Parse(queueAttributes.ApproximateNumberOfMessages),
                    Unit = StandardUnit.Count,
                    Timestamp = DateTime.UtcNow
                },
                new MetricDatum
                {
                    MetricName = "InFlightMessages",
                    Value = double.Parse(queueAttributes.ApproximateNumberOfMessagesNotVisible),
                    Unit = StandardUnit.Count,
                    Timestamp = DateTime.UtcNow
                }
            }
        }, ct);
    }
}
```

### Alerts
- Queue depth > 500 for 10 minutes
- Message age > 5 minutes
- DLQ depth > 10
- Consumer lag increasing
- Publish/consume rate imbalance

## Performance Optimization

### Batching
- Send messages in batches (up to 10)
- Receive messages in batches (up to 10)
- Reduces API calls and costs
- Improves throughput

### Long Polling
- Set WaitTimeSeconds to 20 (maximum)
- Reduces empty receives
- Lowers costs
- Better responsiveness

### Visibility Timeout Tuning
- Set based on typical processing time
- Too short: Duplicate processing
- Too long: Delayed retries
- Recommended: 5 minutes for workflows

### Connection Pooling
```csharp
services.AddAWSService<IAmazonSQS>(new AWSOptions
{
    Credentials = new EnvironmentVariablesAWSCredentials(),
    Region = RegionEndpoint.USEast1,
    DefaultClientConfig = new AmazonSQSConfig
    {
        MaxConnectionsPerServer = 50,
        Timeout = TimeSpan.FromSeconds(30),
        RetryMode = RequestRetryMode.Adaptive
    }
});
```

## Cost Optimization

### Request Bundling
- Use batch operations where possible
- Implement long polling to reduce requests
- Delete messages promptly

### Message Size Optimization
- Keep messages < 64 KB when possible
- Use S3 for large payloads (Extended Client Library)
- Compress large JSON payloads

### Queue Management
- Set appropriate retention period (4 days)
- Clean up DLQ regularly
- Monitor and optimize polling frequency

## Security

### IAM Policies

#### Producer Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:SendMessageBatch",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:workflow-tasks.fifo"
    }
  ]
}
```

#### Consumer Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:ChangeMessageVisibility",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:workflow-tasks.fifo"
    }
  ]
}
```

### Encryption
- Enable server-side encryption (SSE-SQS or SSE-KMS)
- KMS key for sensitive workflow data
- In-transit encryption (TLS 1.3)

### Access Control
- Use VPC endpoints for private access
- Restrict by IAM roles
- Enable CloudTrail logging
- Monitor unauthorized access attempts

## Infrastructure as Code

### Terraform Example
```hcl
resource "aws_sqs_queue" "workflow_tasks" {
  name                        = "workflow-tasks.fifo"
  fifo_queue                  = true
  content_based_deduplication = true
  message_retention_seconds   = 345600  # 4 days
  visibility_timeout_seconds  = 300
  receive_wait_time_seconds   = 20
  
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.workflow_tasks_dlq.arn
    maxReceiveCount     = 3
  })
  
  kms_master_key_id = aws_kms_key.sqs_key.id
  
  tags = {
    Environment = "production"
    Component   = "elsa-workflows"
  }
}

resource "aws_sqs_queue" "workflow_tasks_dlq" {
  name                       = "workflow-tasks-dlq.fifo"
  fifo_queue                 = true
  message_retention_seconds  = 1209600  # 14 days
  
  tags = {
    Environment = "production"
    Component   = "elsa-workflows-dlq"
  }
}
```

## Disaster Recovery

### Queue Backup
- No native backup for SQS
- Implement message archiving before delete
- Store critical messages in S3

### Cross-Region Replication
- Implement application-level replication
- Publish to queues in multiple regions
- Use Route 53 for failover

## Related Components
- [Worker Nodes](worker-nodes.md)
- [Workflow Server](workflow-server.md)
- [Monitoring](monitoring.md)
