# PostgreSQL Database Schema

## Overview
The PostgreSQL database serves as the primary persistence layer for Elsa Workflows, storing workflow definitions, instances, execution state, and all related metadata.

## Database Architecture

### Two-Database Approach

#### 1. Workflow Database (Main)
**Purpose**: Long-term storage of workflow definitions and execution history

**Schema**: `elsa_workflows`

**Key Tables**:
- `workflow_definitions` - Workflow templates and versions
- `workflow_instances` - Active and historical workflow executions
- `activity_execution_records` - Individual activity execution logs
- `bookmarks` - Workflow suspension points
- `triggers` - Workflow start conditions
- `tenants` - Multi-tenancy support
- `users` - User accounts and permissions

#### 2. Runtime State Store
**Purpose**: Transient runtime data and coordination

**Schema**: `elsa_runtime`

**Key Tables**:
- `execution_state_snapshots` - Point-in-time execution state
- `distributed_locks` - Coordination for scheduler
- `worker_registration` - Worker node health tracking
- `task_queue_state` - Queue processing metadata

## Schema Details

### Workflow Definitions Table
```sql
CREATE TABLE workflow_definitions (
    id VARCHAR(255) PRIMARY KEY,
    definition_id VARCHAR(255) NOT NULL,
    version INT NOT NULL,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    is_published BOOLEAN NOT NULL DEFAULT FALSE,
    is_latest BOOLEAN NOT NULL DEFAULT FALSE,
    materialized_workflow JSONB NOT NULL,
    string_data TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    tenant_id VARCHAR(255),
    
    CONSTRAINT uk_workflow_definition_version UNIQUE (definition_id, version),
    INDEX idx_definition_id (definition_id),
    INDEX idx_tenant_id (tenant_id),
    INDEX idx_is_published (is_published),
    INDEX idx_created_at (created_at)
);

-- Partition by tenant for large multi-tenant deployments
-- ALTER TABLE workflow_definitions PARTITION BY LIST (tenant_id);
```

### Workflow Instances Table
```sql
CREATE TABLE workflow_instances (
    id VARCHAR(255) PRIMARY KEY,
    definition_id VARCHAR(255) NOT NULL,
    definition_version INT NOT NULL,
    workflow_state JSONB NOT NULL,
    status VARCHAR(50) NOT NULL,
    sub_status VARCHAR(50),
    correlation_id VARCHAR(255),
    name VARCHAR(500),
    incident_count INT NOT NULL DEFAULT 0,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    finished_at TIMESTAMP,
    faulted_at TIMESTAMP,
    tenant_id VARCHAR(255),
    
    INDEX idx_definition_id (definition_id),
    INDEX idx_correlation_id (correlation_id),
    INDEX idx_status (status),
    INDEX idx_tenant_id (tenant_id),
    INDEX idx_created_at (created_at),
    INDEX idx_finished_at (finished_at)
);

-- Partition by created_at for time-series optimization
CREATE TABLE workflow_instances_2025_q4 PARTITION OF workflow_instances
    FOR VALUES FROM ('2025-10-01') TO ('2026-01-01');
```

### Activity Execution Records Table
```sql
CREATE TABLE activity_execution_records (
    id VARCHAR(255) PRIMARY KEY,
    workflow_instance_id VARCHAR(255) NOT NULL,
    activity_id VARCHAR(255) NOT NULL,
    activity_node_id VARCHAR(255) NOT NULL,
    activity_type VARCHAR(500) NOT NULL,
    activity_type_version INT NOT NULL,
    activity_name VARCHAR(500),
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    has_bookmarks BOOLEAN NOT NULL DEFAULT FALSE,
    status VARCHAR(50) NOT NULL,
    payload JSONB,
    exception TEXT,
    tenant_id VARCHAR(255),
    
    FOREIGN KEY (workflow_instance_id) REFERENCES workflow_instances(id) ON DELETE CASCADE,
    INDEX idx_workflow_instance_id (workflow_instance_id),
    INDEX idx_activity_type (activity_type),
    INDEX idx_status (status),
    INDEX idx_started_at (started_at)
);

-- Partition by started_at for performance
CREATE TABLE activity_execution_records_2025_q4 PARTITION OF activity_execution_records
    FOR VALUES FROM ('2025-10-01') TO ('2026-01-01');
```

### Bookmarks Table
```sql
CREATE TABLE bookmarks (
    id VARCHAR(255) PRIMARY KEY,
    bookmark_id VARCHAR(255) NOT NULL,
    bookmark_hash VARCHAR(255) NOT NULL,
    workflow_instance_id VARCHAR(255) NOT NULL,
    activity_node_id VARCHAR(255) NOT NULL,
    activity_type_name VARCHAR(500) NOT NULL,
    correlation_id VARCHAR(255),
    payload JSONB,
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    tenant_id VARCHAR(255),
    
    FOREIGN KEY (workflow_instance_id) REFERENCES workflow_instances(id) ON DELETE CASCADE,
    INDEX idx_bookmark_hash (bookmark_hash),
    INDEX idx_workflow_instance_id (workflow_instance_id),
    INDEX idx_correlation_id (correlation_id),
    INDEX idx_activity_type (activity_type_name)
);
```

### Triggers Table
```sql
CREATE TABLE triggers (
    id VARCHAR(255) PRIMARY KEY,
    workflow_definition_id VARCHAR(255) NOT NULL,
    workflow_definition_version_id VARCHAR(255) NOT NULL,
    name VARCHAR(500) NOT NULL,
    activity_id VARCHAR(255) NOT NULL,
    hash VARCHAR(255) NOT NULL,
    payload JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    tenant_id VARCHAR(255),
    
    INDEX idx_workflow_definition_id (workflow_definition_id),
    INDEX idx_hash (hash),
    INDEX idx_tenant_id (tenant_id)
);
```

### Distributed Locks Table (Runtime DB)
```sql
CREATE TABLE distributed_locks (
    resource_name VARCHAR(255) PRIMARY KEY,
    lock_id VARCHAR(255) NOT NULL,
    locked_by VARCHAR(255) NOT NULL,
    locked_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,
    
    INDEX idx_expires_at (expires_at)
);

-- Function to acquire lock
CREATE OR REPLACE FUNCTION acquire_lock(
    p_resource_name VARCHAR(255),
    p_lock_id VARCHAR(255),
    p_locked_by VARCHAR(255),
    p_duration_seconds INT
) RETURNS BOOLEAN AS $$
BEGIN
    -- Clean up expired locks
    DELETE FROM distributed_locks 
    WHERE resource_name = p_resource_name AND expires_at < NOW();
    
    -- Try to acquire lock
    INSERT INTO distributed_locks (resource_name, lock_id, locked_by, expires_at)
    VALUES (p_resource_name, p_lock_id, p_locked_by, NOW() + (p_duration_seconds || ' seconds')::INTERVAL)
    ON CONFLICT (resource_name) DO NOTHING;
    
    -- Return true if we acquired the lock
    RETURN EXISTS (
        SELECT 1 FROM distributed_locks 
        WHERE resource_name = p_resource_name AND lock_id = p_lock_id
    );
END;
$$ LANGUAGE plpgsql;
```

### Execution State Snapshots (Runtime DB)
```sql
CREATE TABLE execution_state_snapshots (
    id VARCHAR(255) PRIMARY KEY,
    workflow_instance_id VARCHAR(255) NOT NULL,
    snapshot_data JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_workflow_instance_id (workflow_instance_id),
    INDEX idx_created_at (created_at)
);

-- Auto-cleanup old snapshots (keep last 24 hours)
CREATE OR REPLACE FUNCTION cleanup_old_snapshots() RETURNS void AS $$
BEGIN
    DELETE FROM execution_state_snapshots 
    WHERE created_at < NOW() - INTERVAL '24 hours';
END;
$$ LANGUAGE plpgsql;
```

## Indexes and Performance

### Composite Indexes
```sql
-- Workflow instance lookups
CREATE INDEX idx_workflow_instances_tenant_status 
    ON workflow_instances(tenant_id, status);

CREATE INDEX idx_workflow_instances_definition_status 
    ON workflow_instances(definition_id, status);

-- Activity record queries
CREATE INDEX idx_activity_records_workflow_time 
    ON activity_execution_records(workflow_instance_id, started_at);

-- Bookmark lookups
CREATE INDEX idx_bookmarks_hash_tenant 
    ON bookmarks(bookmark_hash, tenant_id);
```

### JSONB Indexes
```sql
-- Index specific JSONB fields for fast queries
CREATE INDEX idx_workflow_state_variables 
    ON workflow_instances USING GIN ((workflow_state->'variables'));

CREATE INDEX idx_activity_payload_order_id 
    ON activity_execution_records((payload->>'orderId'));
```

## Partitioning Strategy

### Time-Based Partitioning
```sql
-- Partition workflow instances by month
CREATE TABLE workflow_instances (
    -- columns as above
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE workflow_instances_2025_11 PARTITION OF workflow_instances
    FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');

CREATE TABLE workflow_instances_2025_12 PARTITION OF workflow_instances
    FOR VALUES FROM ('2025-12-01') TO ('2026-01-01');

-- Automate partition creation
CREATE OR REPLACE FUNCTION create_monthly_partitions() RETURNS void AS $$
DECLARE
    partition_date DATE;
    partition_name TEXT;
    start_date TEXT;
    end_date TEXT;
BEGIN
    FOR i IN 0..2 LOOP
        partition_date := DATE_TRUNC('month', NOW() + (i || ' months')::INTERVAL);
        partition_name := 'workflow_instances_' || TO_CHAR(partition_date, 'YYYY_MM');
        start_date := TO_CHAR(partition_date, 'YYYY-MM-DD');
        end_date := TO_CHAR(partition_date + INTERVAL '1 month', 'YYYY-MM-DD');
        
        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS %I PARTITION OF workflow_instances FOR VALUES FROM (%L) TO (%L)',
            partition_name, start_date, end_date
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### Tenant-Based Partitioning
```sql
-- For large multi-tenant deployments
CREATE TABLE workflow_definitions (
    -- columns as above
) PARTITION BY LIST (tenant_id);

CREATE TABLE workflow_definitions_tenant_001 PARTITION OF workflow_definitions
    FOR VALUES IN ('tenant-001');

CREATE TABLE workflow_definitions_tenant_002 PARTITION OF workflow_definitions
    FOR VALUES IN ('tenant-002');
```

## Data Retention

### Archive Strategy
```sql
-- Move old workflow instances to archive table
CREATE TABLE workflow_instances_archive (LIKE workflow_instances INCLUDING ALL);

-- Archive completed workflows older than 90 days
INSERT INTO workflow_instances_archive
SELECT * FROM workflow_instances
WHERE status IN ('Completed', 'Cancelled', 'Faulted')
  AND finished_at < NOW() - INTERVAL '90 days';

DELETE FROM workflow_instances
WHERE id IN (SELECT id FROM workflow_instances_archive);
```

### Automatic Cleanup Job
```sql
-- Create cleanup function
CREATE OR REPLACE FUNCTION cleanup_old_data() RETURNS void AS $$
BEGIN
    -- Archive old completed workflows
    WITH archived AS (
        INSERT INTO workflow_instances_archive
        SELECT * FROM workflow_instances
        WHERE status IN ('Completed', 'Cancelled', 'Faulted')
          AND finished_at < NOW() - INTERVAL '90 days'
        RETURNING id
    )
    DELETE FROM workflow_instances WHERE id IN (SELECT id FROM archived);
    
    -- Delete old execution snapshots
    DELETE FROM execution_state_snapshots 
    WHERE created_at < NOW() - INTERVAL '24 hours';
    
    -- Clean up expired locks
    DELETE FROM distributed_locks WHERE expires_at < NOW();
END;
$$ LANGUAGE plpgsql;

-- Schedule daily cleanup (using pg_cron extension)
SELECT cron.schedule('cleanup-old-data', '0 2 * * *', 'SELECT cleanup_old_data()');
```

## High Availability Configuration

### Streaming Replication
```conf
# postgresql.conf (Primary)
wal_level = replica
max_wal_senders = 5
max_replication_slots = 5
synchronous_commit = on
synchronous_standby_names = 'standby1'
```

### Connection Pooling (PgBouncer)
```ini
[databases]
elsa_workflows = host=postgres-primary port=5432 dbname=elsa_workflows
elsa_runtime = host=postgres-primary port=5432 dbname=elsa_runtime

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
reserve_pool_size = 10
reserve_pool_timeout = 5
```

## Backup and Recovery

### Continuous Archiving (WAL)
```bash
# Archive WAL files to S3
archive_command = 'aws s3 cp %p s3://elsa-backups/wal/%f'

# Base backup
pg_basebackup -h postgres-primary -D /backup/base -Fp -Xs -P -R

# Point-in-time recovery
restore_command = 'aws s3 cp s3://elsa-backups/wal/%f %p'
recovery_target_time = '2025-11-18 10:30:00'
```

### Automated Backups
```bash
#!/bin/bash
# Daily backup script
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="elsa_workflows_${BACKUP_DATE}.sql.gz"

pg_dump -h postgres-primary -U elsa_admin elsa_workflows | gzip > ${BACKUP_FILE}
aws s3 cp ${BACKUP_FILE} s3://elsa-backups/daily/

# Retention: Keep 30 days
aws s3 ls s3://elsa-backups/daily/ | awk '{print $4}' | head -n -30 | xargs -I {} aws s3 rm s3://elsa-backups/daily/{}
```

## Monitoring Queries

### Active Workflows
```sql
SELECT 
    status,
    COUNT(*) as count,
    AVG(EXTRACT(EPOCH FROM (NOW() - created_at))) as avg_duration_seconds
FROM workflow_instances
WHERE finished_at IS NULL
GROUP BY status;
```

### Workflow Performance
```sql
SELECT 
    definition_id,
    COUNT(*) as total_executions,
    AVG(EXTRACT(EPOCH FROM (finished_at - created_at))) as avg_duration_seconds,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (finished_at - created_at))) as p95_duration
FROM workflow_instances
WHERE finished_at IS NOT NULL
  AND created_at > NOW() - INTERVAL '24 hours'
GROUP BY definition_id
ORDER BY total_executions DESC
LIMIT 10;
```

### Slow Activities
```sql
SELECT 
    activity_type,
    COUNT(*) as count,
    AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) as avg_duration_seconds,
    MAX(EXTRACT(EPOCH FROM (completed_at - started_at))) as max_duration_seconds
FROM activity_execution_records
WHERE completed_at IS NOT NULL
  AND started_at > NOW() - INTERVAL '1 hour'
GROUP BY activity_type
HAVING AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) > 5
ORDER BY avg_duration_seconds DESC;
```

### Database Size
```sql
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

## Security

### User Roles
```sql
-- Application user (read/write)
CREATE USER elsa_app WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE elsa_workflows TO elsa_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO elsa_app;

-- Worker user (read/write for workers)
CREATE USER elsa_worker WITH PASSWORD 'worker_password';
GRANT CONNECT ON DATABASE elsa_workflows TO elsa_worker;
GRANT SELECT, INSERT, UPDATE ON workflow_instances TO elsa_worker;
GRANT SELECT, INSERT ON activity_execution_records TO elsa_worker;

-- Read-only user (for reporting)
CREATE USER elsa_readonly WITH PASSWORD 'readonly_password';
GRANT CONNECT ON DATABASE elsa_workflows TO elsa_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO elsa_readonly;
```

### Row-Level Security
```sql
-- Enable RLS for multi-tenancy
ALTER TABLE workflow_definitions ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON workflow_definitions
    USING (tenant_id = current_setting('app.current_tenant_id', true));
```

## Related Components
- [Worker Nodes](worker-nodes.md)
- [Workflow Server](workflow-server.md)
- [Database Migrations](database-migrations.md)
