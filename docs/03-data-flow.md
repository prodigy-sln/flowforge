# FlowForge Data Flow and Integration Patterns

## Overview

This document describes how data flows through the FlowForge system, from initial user request to final job completion, including all integration patterns and data transformations.

## Job Submission Flow

```mermaid
sequenceDiagram
    participant User
    participant WebUI
    participant API
    participant Auth
    participant JobMgr as Job Manager
    participant Queue
    participant DB
    participant Cache
    
    User->>WebUI: Submit job request
    WebUI->>API: POST /api/v1/jobs
    API->>Auth: Validate JWT token
    Auth-->>API: Token valid
    
    API->>API: Validate job parameters
    API->>DB: Create job record
    DB-->>API: Job ID
    
    API->>JobMgr: Submit job
    JobMgr->>Queue: Enqueue job
    JobMgr->>Cache: Set job status: queued
    
    API-->>WebUI: Job ID + initial status
    WebUI-->>User: Job submitted
    
    WebUI->>API: Subscribe to job updates
    API->>WebUI: WebSocket connection established
```

## Job Execution Flow

```mermaid
sequenceDiagram
    participant Queue
    participant Worker
    participant Container
    participant Git
    participant Claude
    participant Storage
    participant JobMgr as Job Manager
    participant WS as WebSocket
    
    Queue->>Worker: Job assignment
    Worker->>Container: Start container
    Container->>Container: Initialize environment
    
    Container->>Git: Clone repository
    Git-->>Container: Repository ready
    
    Container->>Claude: Execute task
    Claude-->>Container: Task output
    
    Container->>Git: Commit changes
    Container->>Git: Rebase on target
    
    alt Rebase conflict
        Git-->>Container: Conflict detected
        Container->>Claude: Resolve conflicts
        Claude-->>Container: Resolution
        Container->>Git: Apply resolution
    end
    
    Container->>Git: Push changes
    Container->>Storage: Upload artifacts
    
    Container->>JobMgr: Job complete
    JobMgr->>WS: Notify completion
    JobMgr->>Queue: Acknowledge job
```

## Real-time Updates Flow

```mermaid
graph TD
    subgraph "Container"
        A[Job Execution] -->|Log Entry| B[Log Forwarder]
        A -->|Status Update| C[Status Reporter]
    end
    
    subgraph "Infrastructure"
        B --> D[Message Queue]
        C --> E[Redis Pub/Sub]
    end
    
    subgraph "API Layer"
        D --> F[Log Processor]
        E --> G[Status Processor]
        F --> H[WebSocket Manager]
        G --> H
    end
    
    subgraph "Client"
        H -->|WebSocket| I[Web UI]
        H -->|WebSocket| J[CLI]
    end
```

## Data Transformation Pipeline

### 1. Input Validation and Normalization

```typescript
// Job submission request
interface JobRequest {
  repositoryId: string;
  branch: string;
  targetBranch: string;
  task: string;
  environment?: Record<string, string>;
  priority?: 'low' | 'normal' | 'high';
}

// Internal job representation
interface InternalJob {
  id: UUID;
  userId: UUID;
  repository: {
    id: UUID;
    url: string;
    credentials: EncryptedCredentials;
  };
  execution: {
    branch: string;
    targetBranch: string;
    task: string;
    environment: Record<string, string>;
  };
  metadata: {
    priority: number;
    createdAt: Date;
    ttl: number;
    retryCount: number;
  };
}
```

### 2. Git Operations Data Flow

```mermaid
stateDiagram-v2
    [*] --> Cloning: Start
    Cloning --> BranchCreation: Repository ready
    BranchCreation --> TaskExecution: Branch created
    TaskExecution --> CommitChanges: Task complete
    CommitChanges --> Rebasing: Changes committed
    
    Rebasing --> ConflictDetection: Start rebase
    ConflictDetection --> ConflictResolution: Conflicts found
    ConflictDetection --> PushChanges: No conflicts
    ConflictResolution --> ContinueRebase: Resolved
    ContinueRebase --> PushChanges: Rebase complete
    
    PushChanges --> [*]: Success
    
    Cloning --> [*]: Error
    TaskExecution --> [*]: Error
    ConflictResolution --> [*]: Error
    PushChanges --> [*]: Error
```

### 3. Claude Integration Data Flow

```python
# Conflict resolution prompt construction
def build_conflict_prompt(conflict: GitConflict) -> ClaudePrompt:
    return ClaudePrompt(
        system_message=CONFLICT_RESOLUTION_SYSTEM_PROMPT,
        context={
            "file_path": conflict.file_path,
            "base_branch": conflict.base_branch,
            "current_branch": conflict.current_branch,
            "surrounding_code": extract_context(conflict),
        },
        conflict_markers=conflict.raw_content,
        instructions=[
            "Analyze the conflict",
            "Understand the intent of both changes",
            "Provide a merged solution",
            "Ensure code correctness"
        ]
    )

# Response processing
def process_claude_response(response: ClaudeResponse) -> Resolution:
    return Resolution(
        content=extract_code_block(response.text),
        confidence=calculate_confidence(response),
        explanation=extract_explanation(response.text)
    )
```

## Event-Driven Architecture

### Event Types

```typescript
enum EventType {
  // Job lifecycle events
  JOB_CREATED = 'job.created',
  JOB_QUEUED = 'job.queued',
  JOB_STARTED = 'job.started',
  JOB_PROGRESS = 'job.progress',
  JOB_COMPLETED = 'job.completed',
  JOB_FAILED = 'job.failed',
  JOB_CANCELLED = 'job.cancelled',
  
  // Git events
  GIT_CLONE_STARTED = 'git.clone.started',
  GIT_CLONE_COMPLETED = 'git.clone.completed',
  GIT_CONFLICT_DETECTED = 'git.conflict.detected',
  GIT_CONFLICT_RESOLVED = 'git.conflict.resolved',
  GIT_PUSH_COMPLETED = 'git.push.completed',
  
  // System events
  WORKER_STARTED = 'worker.started',
  WORKER_STOPPED = 'worker.stopped',
  RATE_LIMIT_EXCEEDED = 'rate_limit.exceeded'
}
```

### Event Flow

```mermaid
graph LR
    subgraph "Event Producers"
        A[Job Manager]
        B[Git Service]
        C[Worker Container]
        D[Claude Service]
    end
    
    subgraph "Event Bus"
        E[RabbitMQ/Kafka]
    end
    
    subgraph "Event Consumers"
        F[Notification Service]
        G[Analytics Service]
        H[Audit Logger]
        I[WebSocket Service]
        J[Monitoring Service]
    end
    
    A --> E
    B --> E
    C --> E
    D --> E
    
    E --> F
    E --> G
    E --> H
    E --> I
    E --> J
```

## Caching Strategy

### Cache Layers

```mermaid
graph TD
    A[Client Request] --> B{CDN Cache?}
    B -->|Hit| C[Return Cached]
    B -->|Miss| D{API Cache?}
    D -->|Hit| E[Return from API Cache]
    D -->|Miss| F{Redis Cache?}
    F -->|Hit| G[Return from Redis]
    F -->|Miss| H[Database Query]
    H --> I[Update Redis]
    I --> J[Update API Cache]
    J --> K[Return Response]
```

### Cache Patterns

```python
# Cache-aside pattern for job data
async def get_job(job_id: str) -> Job:
    # Try cache first
    cached = await redis.get(f"job:{job_id}")
    if cached:
        return Job.parse_raw(cached)
    
    # Load from database
    job = await db.get_job(job_id)
    if job:
        # Cache for 1 hour
        await redis.setex(
            f"job:{job_id}", 
            3600, 
            job.json()
        )
    
    return job

# Write-through cache for status updates
async def update_job_status(job_id: str, status: str):
    # Update database
    await db.update_job_status(job_id, status)
    
    # Update cache
    await redis.hset(
        f"job:{job_id}", 
        "status", 
        status
    )
    
    # Publish event
    await redis.publish(
        f"job.status.{job_id}", 
        status
    )
```

## Security Data Flow

### Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant Auth as Auth Service
    participant DB
    
    User->>Frontend: Login credentials
    Frontend->>API: POST /auth/login
    API->>Auth: Validate credentials
    Auth->>DB: Check user
    DB-->>Auth: User data
    Auth->>Auth: Generate JWT
    Auth-->>API: JWT + Refresh token
    API-->>Frontend: Tokens
    Frontend->>Frontend: Store in secure storage
    
    Note over Frontend: Subsequent requests
    Frontend->>API: Request + JWT in header
    API->>Auth: Validate JWT
    Auth-->>API: Claims
    API->>API: Process request
```

### Secret Management Flow

```mermaid
graph TD
    A[Environment Variable] --> B{Secret Type}
    B -->|API Key| C[Kubernetes Secret]
    B -->|Git Credentials| D[Vault]
    B -->|Database Password| E[Cloud KMS]
    
    C --> F[Container Env]
    D --> G[Dynamic Injection]
    E --> H[Connection String]
    
    F --> I[Application]
    G --> I
    H --> I
```

## Error Handling and Recovery

### Retry Logic

```typescript
interface RetryPolicy {
  maxAttempts: number;
  backoffType: 'exponential' | 'linear';
  initialDelay: number;
  maxDelay: number;
  retryableErrors: string[];
}

const jobRetryPolicy: RetryPolicy = {
  maxAttempts: 3,
  backoffType: 'exponential',
  initialDelay: 1000,
  maxDelay: 30000,
  retryableErrors: [
    'NETWORK_ERROR',
    'RATE_LIMIT',
    'CONTAINER_START_FAILED',
    'GIT_CLONE_FAILED'
  ]
};
```

### Error Propagation

```mermaid
graph TD
    A[Container Error] --> B{Error Type}
    B -->|Retryable| C[Add to Retry Queue]
    B -->|Fatal| D[Mark Job Failed]
    B -->|Partial Success| E[Save Progress]
    
    C --> F[Increment Retry Count]
    F --> G{Max Retries?}
    G -->|No| H[Schedule Retry]
    G -->|Yes| D
    
    D --> I[Notify User]
    E --> J[Create Recovery Point]
    J --> K[Allow Resume]
```

## Performance Optimization

### Batch Processing

```python
# Batch job status updates
async def batch_update_statuses():
    batch = []
    async for msg in redis.subscribe("job.status.*"):
        batch.append(msg)
        
        if len(batch) >= 100 or time_since_last() > 1:
            async with db.transaction():
                for update in batch:
                    await db.update_job_status(
                        update.job_id, 
                        update.status
                    )
            batch.clear()
```

### Pipeline Optimization

```mermaid
graph LR
    subgraph "Sequential (Slow)"
        A1[Clone] --> A2[Install] --> A3[Build] --> A4[Test]
    end
    
    subgraph "Parallel (Fast)"
        B1[Clone]
        B1 --> B2[Install]
        B1 --> B3[Download Cache]
        B2 --> B4[Build]
        B3 --> B4
        B4 --> B5[Test]
    end
```

## Monitoring Data Flow

### Metrics Collection

```mermaid
graph TD
    subgraph "Application"
        A[API Service] --> M1[Metrics]
        B[Job Manager] --> M2[Metrics]
        C[Worker] --> M3[Metrics]
    end
    
    subgraph "Collection"
        M1 --> D[Prometheus]
        M2 --> D
        M3 --> D
    end
    
    subgraph "Storage"
        D --> E[Time Series DB]
    end
    
    subgraph "Visualization"
        E --> F[Grafana]
        E --> G[Alerts]
    end
```

### Log Aggregation

```yaml
# Fluentd configuration
<source>
  @type forward
  port 24224
</source>

<filter app.**>
  @type record_transformer
  <record>
    service ${tag_parts[1]}
    environment ${ENV}
    cluster ${CLUSTER_NAME}
  </record>
</filter>

<match app.**>
  @type elasticsearch
  host elasticsearch.flowforge.local
  port 9200
  index_name flowforge-%Y.%m.%d
  <buffer>
    flush_interval 10s
    chunk_limit_size 5M
  </buffer>
</match>
```

## Data Retention and Archival

### Retention Policy

```sql
-- Archive completed jobs older than 30 days
INSERT INTO jobs_archive 
SELECT * FROM jobs 
WHERE completed_at < NOW() - INTERVAL '30 days'
  AND status IN ('success', 'failed', 'cancelled');

-- Clean up old logs
DELETE FROM job_logs 
WHERE created_at < NOW() - INTERVAL '7 days';

-- Compress and move artifacts to cold storage
UPDATE job_artifacts 
SET storage_class = 'GLACIER',
    compressed = true
WHERE created_at < NOW() - INTERVAL '90 days';
```