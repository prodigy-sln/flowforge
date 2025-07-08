# FlowForge Deployment Architecture

## Overview

This document describes the deployment architecture for FlowForge across different environments, from local development to production-scale deployments.

## Environment Tiers

### Local Development

```mermaid
graph TD
    subgraph "Developer Machine"
        IDE[IDE/Editor]
        
        subgraph "Docker Compose"
            API[API Service]
            Worker[Worker Container]
            DB[(PostgreSQL)]
            Redis[(Redis)]
            UI[Web UI]
        end
        
        subgraph "Optional Services"
            Vault[Vault Dev]
            Monitoring[Prometheus/Grafana]
        end
    end
    
    IDE --> API
    API --> DB
    API --> Redis
    Worker --> Redis
    UI --> API
```

**docker-compose.dev.yml:**
```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: flowforge
      POSTGRES_USER: flowforge
      POSTGRES_PASSWORD: devpassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  api:
    build:
      context: ./api
      target: development
    volumes:
      - ./api:/app
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      DATABASE_URL: postgresql://flowforge:devpassword@postgres/flowforge
      REDIS_URL: redis://redis:6379
      JWT_SECRET: devsecret
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
    command: air # Hot reload

  worker:
    build:
      context: ./worker
      target: development
    volumes:
      - ./worker:/app
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      DATABASE_URL: postgresql://flowforge:devpassword@postgres/flowforge
      REDIS_URL: redis://redis:6379
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    depends_on:
      - postgres
      - redis

  ui:
    build:
      context: ./ui
      target: development
    volumes:
      - ./ui:/app
      - /app/node_modules
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8080
    ports:
      - "3000:3000"
    command: npm run dev

volumes:
  postgres_data:
  redis_data:
```

### Staging Environment

```mermaid
graph TD
    subgraph "Cloud Provider"
        subgraph "Kubernetes Cluster"
            subgraph "Ingress"
                LB[Load Balancer]
                Nginx[Nginx Ingress]
            end
            
            subgraph "Application Namespace"
                API[API Deployment]
                Worker[Worker Deployment]
                UI[UI Deployment]
            end
            
            subgraph "Data Namespace"
                PG[(PostgreSQL)]
                Redis[(Redis)]
            end
            
            subgraph "System Namespace"
                Prometheus[Prometheus]
                Grafana[Grafana]
                Loki[Loki]
            end
        end
        
        subgraph "Managed Services"
            S3[S3 Bucket]
            Vault[Vault]
        end
    end
    
    LB --> Nginx
    Nginx --> API
    Nginx --> UI
    API --> PG
    API --> Redis
    Worker --> Redis
    API --> S3
    API --> Vault
```

### Production Environment

```mermaid
graph TD
    subgraph "Multi-Region Setup"
        subgraph "Region 1 (Primary)"
            CDN1[CloudFront]
            ALB1[Application Load Balancer]
            
            subgraph "EKS Cluster 1"
                NG1[Node Group 1]
                NG2[Node Group 2]
                NG3[Node Group 3 - Workers]
            end
            
            RDS1[(RDS Primary)]
            ElastiCache1[(ElastiCache)]
        end
        
        subgraph "Region 2 (DR)"
            CDN2[CloudFront]
            ALB2[Application Load Balancer]
            
            subgraph "EKS Cluster 2"
                NG4[Node Group 4]
                NG5[Node Group 5]
            end
            
            RDS2[(RDS Read Replica)]
            ElastiCache2[(ElastiCache)]
        end
        
        subgraph "Global Services"
            Route53[Route 53]
            S3[S3 + Cross-Region Replication]
            DynamoDB[DynamoDB Global Tables]
        end
    end
    
    Route53 --> CDN1
    Route53 --> CDN2
    CDN1 --> ALB1
    CDN2 --> ALB2
    RDS1 -.-> RDS2
    S3 -.-> S3
```

## Kubernetes Architecture

### Namespace Structure

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: flowforge-prod
  labels:
    name: flowforge-prod
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: flowforge-workers
  labels:
    name: flowforge-workers
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: flowforge-data
  labels:
    name: flowforge-data
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: flowforge-monitoring
  labels:
    name: flowforge-monitoring
    environment: production
```

### API Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flowforge-api
  namespace: flowforge-prod
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: flowforge-api
  template:
    metadata:
      labels:
        app: flowforge-api
    spec:
      serviceAccountName: flowforge-api
      containers:
      - name: api
        image: flowforge/api:1.0.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: flowforge-db-secret
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: flowforge-redis-secret
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
---
apiVersion: v1
kind: Service
metadata:
  name: flowforge-api
  namespace: flowforge-prod
spec:
  selector:
    app: flowforge-api
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  type: ClusterIP
```

### Worker Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flowforge-worker
  namespace: flowforge-workers
spec:
  replicas: 5
  selector:
    matchLabels:
      app: flowforge-worker
  template:
    metadata:
      labels:
        app: flowforge-worker
    spec:
      serviceAccountName: flowforge-worker
      containers:
      - name: worker
        image: flowforge/worker:1.0.0
        env:
        - name: WORKER_CONCURRENCY
          value: "2"
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: claude-secret
              key: api-key
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock
        securityContext:
          privileged: true  # Required for Docker-in-Docker
      volumes:
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
          type: Socket
      nodeSelector:
        workload-type: worker
      tolerations:
      - key: worker-only
        operator: Equal
        value: "true"
        effect: NoSchedule
```

### Horizontal Pod Autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flowforge-api-hpa
  namespace: flowforge-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flowforge-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: flowforge-prod
spec:
  podSelector:
    matchLabels:
      app: flowforge-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector:
        matchLabels:
          app: flowforge-ui
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: flowforge-data
    ports:
    - protocol: TCP
      port: 5432  # PostgreSQL
    - protocol: TCP
      port: 6379  # Redis
  - to:
    - namespaceSelector:
        matchLabels:
          name: flowforge-workers
  - ports:
    - port: 443   # HTTPS for external APIs
      protocol: TCP
    - port: 53    # DNS
      protocol: UDP
```

## CI/CD Pipeline

### GitOps with ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flowforge-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/flowforge/deployments
    targetRevision: main
    path: environments/production
  destination:
    server: https://kubernetes.default.svc
    namespace: flowforge-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - Validate=true
    - CreateNamespace=false
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Blue-Green Deployment

```mermaid
graph LR
    subgraph "Current State"
        LB[Load Balancer]
        Blue[Blue Environment v1.0]
        Green[Green Environment - Idle]
        
        LB --> Blue
    end
    
    subgraph "Deploy New Version"
        LB2[Load Balancer]
        Blue2[Blue Environment v1.0]
        Green2[Green Environment v1.1]
        
        LB2 -.-> Blue2
        LB2 --> Green2
    end
    
    subgraph "Switch Traffic"
        LB3[Load Balancer]
        Blue3[Blue Environment - Idle]
        Green3[Green Environment v1.1]
        
        LB3 --> Green3
    end
```

## Disaster Recovery

### Backup Strategy

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
  namespace: flowforge-data
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: flowforge/backup-tool:latest
            command:
            - /bin/sh
            - -c
            - |
              pg_dump $DATABASE_URL | gzip > backup-$(date +%Y%m%d-%H%M%S).sql.gz
              aws s3 cp backup-*.sql.gz s3://flowforge-backups/postgres/
              redis-cli --rdb /tmp/redis-backup.rdb
              aws s3 cp /tmp/redis-backup.rdb s3://flowforge-backups/redis/
            env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: postgres-backup
                  key: url
          restartPolicy: OnFailure
```

### Multi-Region Failover

```mermaid
sequenceDiagram
    participant User
    participant Route53
    participant HealthCheck
    participant Primary
    participant Secondary
    
    User->>Route53: Request
    Route53->>HealthCheck: Check primary health
    
    alt Primary Healthy
        HealthCheck-->>Route53: Healthy
        Route53->>Primary: Route to primary
        Primary-->>User: Response
    else Primary Unhealthy
        HealthCheck-->>Route53: Unhealthy
        Route53->>Secondary: Route to secondary
        Note over Secondary: Promote read replica
        Secondary-->>User: Response
    end
```

## Security Hardening

### Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: flowforge-prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### RBAC Configuration

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: flowforge-api-role
  namespace: flowforge-prod
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["create", "get", "list", "watch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: flowforge-api-rolebinding
  namespace: flowforge-prod
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: flowforge-api-role
subjects:
- kind: ServiceAccount
  name: flowforge-api
  namespace: flowforge-prod
```

## Monitoring and Observability

### Prometheus Configuration

```yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: flowforge-api
  namespace: flowforge-monitoring
spec:
  selector:
    matchLabels:
      app: flowforge-api
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    scrapeTimeout: 10s
```

### Grafana Dashboards

```json
{
  "dashboard": {
    "title": "FlowForge Operations",
    "panels": [
      {
        "title": "Job Processing Rate",
        "targets": [
          {
            "expr": "rate(flowforge_jobs_processed_total[5m])"
          }
        ]
      },
      {
        "title": "API Response Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
          }
        ]
      }
    ]
  }
}
```

## Cost Optimization

### Node Configuration

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: flowforge-production
  region: us-east-1

nodeGroups:
  - name: core-services
    instanceType: t3.medium
    desiredCapacity: 3
    minSize: 3
    maxSize: 6
    spot: false
    labels:
      workload-type: core

  - name: worker-nodes
    instanceType: c5.2xlarge
    desiredCapacity: 2
    minSize: 0
    maxSize: 20
    spot: true
    spotInstancePools: 3
    labels:
      workload-type: worker
    taints:
      - key: worker-only
        value: "true"
        effect: NoSchedule
```

### Resource Optimization

```yaml
apiVersion: v1
kind: VerticalPodAutoscaler
metadata:
  name: flowforge-api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flowforge-api
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: api
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 2Gi
```

## Deployment Checklist

### Pre-Deployment
- [ ] All tests passing
- [ ] Security scan completed
- [ ] Performance benchmarks met
- [ ] Documentation updated
- [ ] Database migrations prepared
- [ ] Rollback plan documented

### Deployment Steps
1. Update configuration in Git
2. ArgoCD syncs changes
3. Monitor deployment progress
4. Run smoke tests
5. Monitor metrics and logs
6. Update status page

### Post-Deployment
- [ ] Verify all services healthy
- [ ] Check error rates
- [ ] Monitor performance metrics
- [ ] User acceptance testing
- [ ] Update release notes

## Emergency Procedures

### Rollback Process
```bash
# Immediate rollback
kubectl rollout undo deployment/flowforge-api -n flowforge-prod

# Rollback to specific revision
kubectl rollout undo deployment/flowforge-api --to-revision=42 -n flowforge-prod

# Database rollback
kubectl apply -f emergency/database-restore-job.yaml
```

### Incident Response
1. Acknowledge incident
2. Assess impact
3. Implement fix or rollback
4. Monitor recovery
5. Post-mortem analysis