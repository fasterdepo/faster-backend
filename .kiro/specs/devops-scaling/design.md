# DevOps & Scaling - Design Document

## Overview

This design establishes a production-grade, multi-region, highly available infrastructure for BANKERCHANGER on AWS. The architecture prioritizes zero-downtime deployments, automatic failover, comprehensive observability, and horizontal scalability. All infrastructure is defined as code using Terraform, with containerized services orchestrated via ECS on Fargate.

### Key Design Principles

1. **Infrastructure as Code**: All resources defined in Terraform for reproducibility and version control
2. **High Availability**: Multi-AZ deployment with automatic failover and no single points of failure
3. **Horizontal Scalability**: Stateless services that scale based on demand
4. **Observability**: Comprehensive distributed tracing, centralized logging, and metrics
5. **Security**: Defense in depth with encryption, WAF, DDoS protection, and network isolation
6. **Disaster Recovery**: Automated backups with point-in-time recovery and multi-region replication

---

## Architecture Overview

### Regional Topology

**Primary Region: us-east-1 (US-East)**
- Production workloads
- Primary database
- Primary Redis cluster
- Main CDN origin

**Secondary Region: eu-west-1 (EU-West)**
- Read replicas for analytics
- Warm standby capability
- Regional CDN distribution

**Global Distribution**
- CloudFront CDN for static assets and caching
- Route53 health checks and DNS failover
- ALB in primary region with NLB for non-HTTP protocols

### VPC Architecture

```
Primary VPC (10.0.0.0/16)
├── Public Subnets (3 AZs)
│   ├── AZ-A: 10.0.1.0/24
│   ├── AZ-B: 10.0.2.0/24
│   └── AZ-C: 10.0.3.0/24
├── Private Subnets (3 AZs)
│   ├── AZ-A: 10.0.11.0/24 (ECS, Kubernetes)
│   ├── AZ-B: 10.0.12.0/24 (ECS, Kubernetes)
│   └── AZ-C: 10.0.13.0/24 (ECS, Kubernetes)
├── Database Subnets (3 AZs)
│   ├── AZ-A: 10.0.21.0/24 (RDS)
│   ├── AZ-B: 10.0.22.0/24 (RDS)
│   └── AZ-C: 10.0.23.0/24 (RDS)
└── Cache Subnets (3 AZs)
    ├── AZ-A: 10.0.31.0/24 (Redis)
    ├── AZ-B: 10.0.32.0/24 (Redis)
    └── AZ-C: 10.0.33.0/24 (Redis)

Secondary VPC (10.1.0.0/16) - similar structure
```

### Security Groups

1. **ALB Security Group**
   - Ingress: HTTP (80), HTTPS (443) from 0.0.0.0/0
   - Egress: All to ECS security group

2. **ECS Security Group**
   - Ingress: 8080 from ALB, 3000 from ECS (WebSocket)
   - Egress: All to RDS, ElastiCache, external APIs

3. **RDS Security Group**
   - Ingress: 5432 from ECS security group and RDS Proxy
   - Egress: None

4. **ElastiCache Security Group**
   - Ingress: 6379 from ECS security group
   - Egress: None

5. **Bastion Security Group**
   - Ingress: 22 from company VPN only
   - Egress: All

---

## Terraform Module Structure

### Directory Organization

```
infrastructure/
├── terraform/
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── main.tf
│   │   │   ├── terraform.tfvars
│   │   │   └── backend.tf
│   │   ├── staging/
│   │   │   ├── main.tf
│   │   │   ├── terraform.tfvars
│   │   │   └── backend.tf
│   │   └── prod/
│   │       ├── main.tf
│   │       ├── terraform.tfvars
│   │       └── backend.tf
│   ├── modules/
│   │   ├── vpc/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── versions.tf
│   │   ├── rds/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── versions.tf
│   │   ├── elasticache/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── versions.tf
│   │   ├── ecs/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── versions.tf
│   │   ├── alb/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── versions.tf
│   │   ├── observability/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── versions.tf
│   │   ├── backup/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── versions.tf
│   │   ├── security/
│   │   │   ├── main.tf (WAF, DDoS, KMS)
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── versions.tf
│   │   └── dns/
│   │       ├── main.tf (Route53, DNS failover)
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── versions.tf
│   ├── globals.tf (data sources, locals)
│   ├── variables.tf (root variables)
│   ├── outputs.tf (root outputs)
│   ├── versions.tf (provider versions)
│   └── state.tf (S3 backend config)
└── README.md
```

### Module Responsibilities

**vpc module**
- VPC creation with public/private/database/cache subnets
- NAT gateways for private subnet egress
- VPC endpoints for AWS services
- Internet gateways and routing tables

**rds module**
- RDS cluster (Aurora PostgreSQL) in multi-AZ
- Parameter groups and subnet groups
- Enhanced monitoring and backup configuration
- RDS Proxy for connection pooling
- Read replicas in secondary region

**elasticache module**
- Redis cluster with 3+ nodes
- Cluster mode enabled for sharding
- Automatic failover and maintenance windows
- Parameter groups and subnet groups
- CloudWatch alarms for memory/CPU

**ecs module**
- ECS cluster definition
- Task definitions for backend, indexer, frontend
- Service definitions with load balancer integration
- Auto scaling groups and scaling policies
- CloudWatch log groups

**alb module**
- Application Load Balancer
- HTTPS/TLS termination
- Target groups for each service
- Health check configuration
- Sticky sessions for WebSocket support

**observability module**
- Jaeger deployment (ECS service)
- CloudWatch Logs integration
- Prometheus deployment (ECS service)
- AlertManager configuration
- Grafana deployment (ECS service)

**backup module**
- AWS Backup plans and vaults
- Lifecycle policies for backup retention
- Cross-region backup replication
- Encryption key management

**security module**
- AWS WAF with rules for OWASP Top 10
- Shield DDoS protection
- KMS keys for encryption at rest
- AWS Secrets Manager for database credentials
- VPC Flow Logs

**dns module**
- Route53 hosted zone
- Health checks for primary/secondary regions
- Failover records
- Weighted routing for multi-region

---

## Container Orchestration: ECS on Fargate

### Rationale for ECS vs Kubernetes

**ECS on Fargate chosen because:**
1. AWS-native service (simpler operations)
2. Serverless container execution (no cluster management)
3. Built-in integration with ALB, CloudWatch, Secrets Manager
4. Lower operational overhead than self-managed Kubernetes
5. Cost-effective for startup at current scale (<10k req/s)

**Alternative: Self-managed Kubernetes**
- Would provide more flexibility and portability
- Higher operational complexity
- Better suited for multi-cloud strategies
- Can be adopted later if needed

### Task Definitions

```
Backend Task Definition
├── Container: bankerchanger-backend
│   ├── Image: <account>.dkr.ecr.us-east-1.amazonaws.com/bankerchanger-backend:latest
│   ├── CPU: 512 (0.25 vCPU)
│   ├── Memory: 1024 MB
│   ├── Port: 8080
│   ├── Environment variables:
│   │   ├── DATABASE_URL (from Secrets Manager)
│   │   ├── REDIS_URL (from Secrets Manager)
│   │   ├── NODE_ENV: production
│   │   └── JAEGER_AGENT_HOST: localhost
│   ├── Logging:
│   │   ├── Driver: awslogs
│   │   └── Group: /ecs/bankerchanger-backend
│   └── Health check:
│       ├── Command: ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
│       ├── Interval: 30 seconds
│       ├── Timeout: 5 seconds
│       ├── Retries: 3
│       └── Start period: 60 seconds

Indexer Task Definition
├── Container: bankerchanger-indexer
│   ├── Image: <account>.dkr.ecr.us-east-1.amazonaws.com/bankerchanger-indexer:latest
│   ├── CPU: 512 (0.25 vCPU)
│   ├── Memory: 2048 MB (higher memory for Stellar ledger indexing)
│   ├── Port: 3001
│   ├── Environment variables:
│   │   ├── DATABASE_URL
│   │   ├── STELLAR_RPC_URL
│   │   ├── JAEGER_AGENT_HOST: localhost
│   │   └── LOG_LEVEL: info
│   ├── Logging: /ecs/bankerchanger-indexer
│   └── No external health check (background job)

Frontend Task Definition
├── Container: bankerchanger-frontend
│   ├── Image: <account>.dkr.ecr.us-east-1.amazonaws.com/bankerchanger-frontend:latest
│   ├── CPU: 256 (0.25 vCPU)
│   ├── Memory: 512 MB
│   ├── Port: 3000
│   ├── Environment variables:
│   │   ├── REACT_APP_API_URL: https://api.bankerchanger.com
│   │   └── REACT_APP_WS_URL: wss://ws.bankerchanger.com
│   ├── Logging: /ecs/bankerchanger-frontend
│   └── Health check: http://localhost:3000/index.html
```

### ECS Service Configuration

**Backend Service**
- Task count (desired): 3 (minimum for HA)
- Launch type: FARGATE
- Placement: SPREAD across AZs
- Networking: ALB target group with path-based routing
- Auto scaling: target tracking CPU 70%, memory 80%
- Min tasks: 2, Max tasks: 10
- Deployment strategy: ECS Rolling update (1 min healthy task count)

**Indexer Service**
- Task count: 1 (single indexer to maintain ledger sync)
- Launch type: FARGATE
- Networking: No ALB, internal only
- Auto scaling: Disabled (single instance requirement)

**Frontend Service**
- Task count: 2
- Launch type: FARGATE
- Networking: CloudFront distribution
- Auto scaling: target tracking CPU 75%
- Min tasks: 2, Max tasks: 5

---

## Horizontal Scaling Strategy

### Scaling Triggers and Metrics

**CPU Utilization**
- Scale up target: 70% average across service
- Scale up cooldown: 300 seconds
- Scale down target: 30% average
- Scale down cooldown: 600 seconds

**Memory Utilization**
- Scale up target: 80% average
- Scale down target: 40% average

**Request Count**
- Target: 1,000 requests per minute per task
- Auto scaling policy: target tracking requests per task

**Custom Metrics**
- WebSocket connections: Scale up if >9,000 per instance
- Database connection pool usage: Alert if >90%
- Queue depth: Scale up if SQS queue depth >1,000

### Auto Scaling Policies

**Target Tracking Scaling (Primary)**
```
Backend Service:
  - Metric: CPU utilization
  - Target: 70%
  - Scale-up adjustment: +2 tasks (up to max)
  - Scale-down adjustment: -1 task (down to min)
  - Cooldown periods: 5 min up, 10 min down

  - Metric: Memory utilization
  - Target: 80%
  - Same adjustment/cooldown as CPU
```

**Step Scaling (Supplemental)**
```
If CPU > 90% for 2 minutes:
  - Add 3 tasks immediately
If CPU > 95% for 1 minute:
  - Add 5 tasks immediately (burst response)
```

### Database Connection Pooling

**RDS Proxy Configuration**
- Session pool size: 150
- Transaction pool size: 150
- Max idle connections: 10
- Connection borrow timeout: 30 seconds
- Session pinning: Enabled for stateful operations
- Initialization query: "SET search_path TO public"

**PgBouncer Alternative (if self-managed)**
```
# pgbouncer.ini
[databases]
bankerchanger = host=rds-endpoint.us-east-1.rds.amazonaws.com port=5432 \
                dbname=bankerchanger user=postgres password=$DATABASE_PASSWORD

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 10
reserve_pool_size = 5
reserve_pool_timeout = 3
max_db_connections = 100
max_user_connections = 50
```

### WebSocket Scaling with Redis Pub/Sub

**Architecture**
```
Client → ALB (sticky session) → Backend Instance 1
                               ↓
                           Redis Cluster
                               ↑
         Client → ALB → Backend Instance 2
```

**Sticky Session Configuration (ALB)**
- Stickiness enabled: Yes
- Duration: 1 day
- Cookie: AWSALB-STICKINESS
- Path pattern: /ws*

**Redis Pub/Sub Implementation**
```
Backend subscribes to channel: transactions:*

When new market/bet:
  redis.publish("transactions:market:123", {
    type: "market_updated",
    data: {...}
  })

Connected WebSocket clients receive update via:
  subscription.on("message", (channel, msg) => {
    ws.send(JSON.stringify(msg))
  })

Max connections per instance: 10,000
Memory overhead: ~1KB per connection
```

**Scaling Logic**
- Monitor: Active WebSocket connections per instance
- If any instance >9,000 connections: Add 1 task
- Target distribution: Evenly distributed via sticky sessions
- Connection limits enforced at application level

---

## Database High Availability & Failover

### RDS Multi-AZ Architecture

```
Primary DB Cluster (us-east-1)
├── Master Instance (us-east-1a)
│   └── Storage: io2 (provisioned IOPS)
│
├── Standby Replica (us-east-1b) - Synchronous
│   └── Automatic failover within 120 seconds
│
└── Read Replica (us-east-1c) - Asynchronous
    └── For analytics queries

Secondary Region (eu-west-1) - Read Replica with 1 min replication lag
```

### Failover Mechanism

**Automatic Failover**
- Monitoring: AWS RDS enhanced monitoring every 10 seconds
- Failure detection: Multi-AZ failover engine monitors primary
- Failover trigger: Master unresponsive for 120 seconds
- Standby promotion: <2 minutes
- DNS update: Automatic via RDS endpoint
- No application code changes needed

**Failover Process**
1. Health check failure detected
2. RDS initiates failover to standby
3. Standby promoted to primary
4. New standby spun up in original AZ
5. DNS record updated (transparent to application)
6. Replication resumes

**Monitoring & Alerts**
- CloudWatch metric: DBInstanceIdentifier.FailoverCount
- CloudWatch alarm: Alert if failover occurs
- Replication lag monitoring: < 1 second target
- Custom metric: Database connection count

### Backup Strategy

**Automated Backups**
- Daily full backup at 02:00 UTC
- Automatic incremental backups every 6 hours
- Retention: 30 days
- Encryption: AES-256 (KMS)
- Backup location: S3 in same region

**Point-in-Time Recovery (PITR)**
- Enabled: Yes
- Retention: 7 days
- Recovery time: ~5-10 minutes
- Recovery point: Any point within 7 days

**Manual Snapshots**
- Before major deployments
- Before data migrations
- Monthly for compliance
- Replicated to secondary region

**Backup Testing**
- Monthly: Restore from snapshot to test instance
- Verify data integrity and schema
- Verify performance characteristics
- Document recovery time

**Cross-Region Replication**
- Read replica in eu-west-1
- Replication lag: <1 second
- Can be promoted to independent DB if primary fails

---

## Redis High Availability & Failover

### Redis Cluster Configuration

```
Redis Cluster (us-east-1)
├── Node 1 (us-east-1a) - Master
│   ├── Slots: 0-5460
│   └── Replica: Node 4
├── Node 2 (us-east-1b) - Master
│   ├── Slots: 5461-10922
│   └── Replica: Node 5
├── Node 3 (us-east-1c) - Master
│   ├── Slots: 10923-16383
│   └── Replica: Node 6
├── Node 4 (us-east-1a) - Replica
├── Node 5 (us-east-1b) - Replica
└── Node 6 (us-east-1c) - Replica

Total: 6 nodes (3 masters + 3 replicas)
Replication factor: 1 (each master has 1 replica)
```

### Failover Configuration

**Automatic Failover (ElastiCache Cluster Mode Enabled)**
- Multi-AZ: Enabled
- Automatic failover: Enabled
- Failover time: <30 seconds
- Detection: Health check every 15 seconds
- Promotion: Replica auto-promoted to master

**Manual Failover Testing**
- Monthly test: Reboot nodes to simulate failure
- Verify: WebSocket connections maintained
- Verify: No data loss
- Verify: Session state preserved

### Persistence Configuration

**RDB (Snapshot) Persistence**
- Snapshot frequency: Every 10 minutes
- Disk space: Requires 2x cluster memory
- Backup location: S3
- Retention: 7 days

**AOF (Append-Only File) Persistence**
- Enabled: Yes
- Fsync frequency: Every second
- File location: /var/lib/redis/appendonly.aof
- Size limit: Rewritten if >100MB

**Durability Tradeoff**
- RDB: Fast recovery, higher data loss risk
- AOF: Slower recovery, lower data loss risk
- Combined: Both enabled for maximum safety

### Memory Management

**Memory Alerts**
- 80% threshold: CloudWatch alarm
- 90% threshold: Auto-evict least-recently-used keys
- Eviction policy: allkeys-lru
- Max memory: Set to 90% of instance memory

**Monitoring Metrics**
- Used memory bytes
- Eviction rate
- Connected clients
- Commands per second
- Network I/O

---

## Observability Stack Architecture

### Distributed Tracing (OpenTelemetry + Jaeger)

**Jaeger Deployment**
- Collector: ECS service in private subnet
- Storage: Elasticsearch backend (3 nodes)
- Query UI: Jaeger UI at monitoring.bankerchanger.com
- Retention: 7 days (configurable)

**Instrumentation**
```
Backend (Express + Node.js):
├── Auto-instrumentation: @opentelemetry/auto
├── Manual spans: DatabasePool, WebSocket operations
├── Propagation: W3C Trace Context headers
└── Sampling: 10% in prod, 100% in dev

Indexer (Node.js):
├── Spans: Stellar RPC calls, database inserts
├── Parent trace: Block processing
└── Error spans: Failed transactions

Frontend (React):
├── Auto-instrumentation: @opentelemetry/browser
├── User interaction spans
├── API call spans with timing
└── Error boundary spans
```

**Trace Flow Example**
```
User clicks "Place Bet" →
  Frontend span: "placeBeT_click" (t=0ms)
    → Backend span: "POST /api/bets" (t=10ms)
      → Span: "validateBet" (t=15ms)
      → Span: "checkUserBalance" → DB query (t=25ms)
      → Span: "createBet" → DB insert (t=35ms)
      → Span: "publishWebSocket" → Redis pub/sub (t=42ms)
    ← Response sent (t=50ms)
  ← WebSocket message received (t=55ms)
```

**Jaeger Query Examples**
- Service: "backend"
- Operation: "POST /api/bets"
- Tags: status=200, error=false
- Duration: >100ms
- Find slow database operations
- Trace full request path

### Centralized Logging (ELK Stack)

**Elasticsearch Cluster**
- Nodes: 3 (1 master, 2 data)
- Instance type: t3.large
- Storage: 1TB per node (EBS)
- Shards: 5 primary + 1 replica
- Index rotation: Daily
- Retention: 30 days

**Logstash Pipeline**
```
Input:
  └── CloudWatch Logs (ECS log groups)

Filter:
  ├── Parse JSON logs (Pino format)
  ├── Extract fields: timestamp, level, service, message
  ├── Parse error stacks
  └── Tag by service type

Output:
  └── Elasticsearch (daily indices)
```

**Kibana Dashboard**
- Index pattern: logs-*
- Visualizations:
  - Error rate by service
  - Error types (top 10)
  - Response times (p50, p95, p99)
  - Service dependencies
- Alerting: Email on error rate spike >1%

**Structured Logging (Pino)**
```javascript
logger.info({
  service: "backend",
  requestId: "abc123",
  userId: "user456",
  action: "placeBet",
  marketId: "market789",
  amount: 100,
  duration: 45,
  status: "success"
})

// Queryable in Kibana: 
// service: backend AND status: failed AND marketId: market789
```

### Metrics & Alerting (Prometheus + Grafana)

**Prometheus Configuration**
```yaml
global:
  scrape_interval: 30s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'backend'
    static_configs:
      - targets: ['backend:9090']
    metrics_path: '/metrics'
  
  - job_name: 'indexer'
    static_configs:
      - targets: ['indexer:9091']
  
  - job_name: 'rds'
    static_configs:
      - targets: ['localhost:9090'] # RDS exporter
  
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-master:9091'] # Redis exporter
```

**Key Metrics**
```
Backend:
  http_request_duration_seconds (histogram)
  http_requests_total (counter)
  db_query_duration_seconds (histogram)
  websocket_connections_active (gauge)
  redis_command_duration_seconds (histogram)

Indexer:
  stellar_ledger_height (gauge)
  stellar_block_processing_duration_seconds (histogram)
  indexer_lag_seconds (gauge)
  db_insert_duration_seconds (histogram)

Infrastructure:
  container_cpu_usage_percent (gauge)
  container_memory_usage_bytes (gauge)
  container_network_bytes_sent (counter)
  rds_database_connections (gauge)
  redis_used_memory_bytes (gauge)
```

**AlertManager Rules**
```yaml
groups:
  - name: backend
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.01
        for: 5m
        annotations:
          summary: "High error rate detected"
      
      - alert: SlowAPI
        expr: http_request_duration_seconds{quantile="0.95"} > 1
        for: 10m
        annotations:
          summary: "API response time > 1s"
      
      - alert: DatabaseDown
        expr: up{job="rds"} == 0
        for: 2m
        annotations:
          summary: "RDS database unreachable"
      
      - alert: HighMemoryUsage
        expr: redis_used_memory_bytes / redis_memory_max_bytes > 0.8
        for: 5m
        annotations:
          summary: "Redis memory > 80%"

  - name: deployment
    rules:
      - alert: HighCPUUtilization
        expr: container_cpu_usage_percent > 85
        for: 3m
        annotations:
          summary: "Container CPU > 85% for 3 minutes"
      
      - alert: HighMemoryUtilization
        expr: container_memory_usage_bytes / container_memory_max_bytes > 0.85
        for: 3m
```

**Grafana Dashboards**
1. **System Overview**: CPU, memory, network across all services
2. **API Performance**: Response times, request rates, error rates
3. **Database Health**: Connections, query times, replication lag
4. **Redis Health**: Memory, connections, command latency
5. **Application Metrics**: Business metrics, user activity
6. **Deployment Status**: Container count, restart count, failure rate

**Alert Routing**
```
AlertManager → Slack (dev channel)
            → PagerDuty (on-call rotation)
            → Email (ops team)

Severity levels:
  critical → PagerDuty immediate + Slack
  warning  → Slack only
  info     → Slack thread
```

---

## Disaster Recovery Strategy

### RTO/RPO Targets

| Failure Scenario | RTO | RPO | Strategy |
|---|---|---|---|
| Availability Zone failure | <2 min | 0 sec | Multi-AZ replication |
| Region failure | <4 hours | <1 hour | Cross-region read replica + backup restore |
| Data corruption | <30 min | <1 hour | Point-in-time recovery |
| Application failure | <5 min | N/A | Blue-green deployment |
| Complete data loss | <4 hours | <24 hours | S3-stored backups |

### Backup Strategy

**Database Backups**
- Frequency: Daily full (02:00 UTC) + hourly incremental
- Retention: 30 days in S3
- Encryption: AES-256 with KMS
- Cross-region: Replicated to eu-west-1
- Testing: Monthly restore test to verify integrity

**Application Configuration Backups**
- Terraform state: Backed up in S3 (versioning enabled)
- Secrets: AWS Secrets Manager (built-in replication)
- Docker images: ECR (cross-region replication)

**Testing & Validation**
```
Monthly Disaster Recovery Drill:
1. Restore database from 30-day-old backup
2. Restore application from tagged Docker image
3. Verify data integrity and schema
4. Run smoke tests
5. Measure actual RTO
6. Document findings and improvements
```

### Disaster Recovery Procedures

**Scenario 1: Single AZ Failure**
- Automatic RDS failover to standby AZ (<2 minutes)
- ECS tasks redistributed to healthy AZs
- No manual intervention required
- Expected impact: None (HA configuration)

**Scenario 2: Regional Outage**
```
Recovery Procedure:
1. Detect: CloudWatch health checks fail, Route53 detects down
2. Failover: Route53 fails over to secondary region
3. Database: Promote read replica to standalone database
4. Application: Deploy from backup image to secondary region
5. DNS: Update Route53 to point to secondary region
6. Verification: Run smoke tests
7. Timeline: ~30 minutes for manual execution
```

**Scenario 3: Data Corruption**
```
Recovery Procedure:
1. Identify: Detect via CloudWatch anomaly detection
2. Isolate: Pause indexer and application writes
3. Restore: Point-in-time recovery to pre-corruption state
4. Verify: Validate data integrity
5. Reindex: Re-run indexer to sync state
6. Resume: Resume application writes
7. Timeline: <30 minutes
```

**Scenario 4: Complete Infrastructure Loss**
```
Recovery Procedure:
1. Provision: Terraform apply in new region
2. Database: Restore from S3 backup
3. Application: Deploy from ECR backup image
4. Configuration: Load from AWS Secrets Manager
5. Verification: Run full test suite
6. DNS: Update Route53
7. Timeline: <4 hours
```

---

## Multi-Region Deployment

### Regional Strategy

**Primary Region: us-east-1**
- Production traffic
- Master database
- Master Redis cluster
- Primary CDN origin
- Full feature set

**Secondary Region: eu-west-1**
- Read replicas for analytics
- Warm standby (can promote to primary)
- Regional CDN caching
- Can serve low-latency for EU users

**Future Tertiary Regions**
- ap-southeast-1 (Singapore) for Asia-Pacific
- ap-northeast-1 (Tokyo) for Japan
- Follows same pattern as eu-west-1

### Route53 Routing Policies

**Primary-Failover Pattern**
```
Routing rule:
  1. Route to us-east-1 ALB (primary)
     Health check: HTTP /health every 30s
     If unhealthy: failover after 3 consecutive failures (90s)
  2. Route to eu-west-1 ALB (secondary)
     Backup target if primary fails
```

**Weighted Distribution (Future)**
```
Route traffic by percentage:
  us-east-1: 90%
  eu-west-1: 10%

Gradually increase eu-west-1 percentage as capacity grows
```

**Geolocation Routing (Future)**
```
Route based on geographic location:
  North America (us-east-1): 
    US, Canada, Mexico
  Europe (eu-west-1):
    All European countries
  Asia-Pacific (future):
    All Asian countries
```

### Database Replication

**Primary → Secondary (Read Replica)**
- Asynchronous replication
- Lag: <1 second typical
- Promotion path: Can be promoted to standalone primary
- Cross-region snapshot backups

**Secondary → Tertiary (if needed)**
- Cascading replicas for additional regions
- Each adds ~1 second latency per hop

### CDN Strategy (CloudFront)

**Static Assets (S3 origin)**
```
CloudFront distribution:
├── Origin 1: S3 bucket (us-east-1)
├── Caching behavior:
│   ├── Path: /assets/*
│   ├── TTL: 1 year for versioned assets
│   ├── TTL: 1 hour for dynamic assets
│   └── Compression: gzip, brotli
├── Edge locations: All (for maximum performance)
└── Custom domain: assets.bankerchanger.com
```

**API Responses (ALB origin)**
```
CloudFront distribution:
├── Origin: ALB (regional)
├── Caching behavior:
│   ├── Path: /api/public/*
│   ├── TTL: 5 minutes (cacheable endpoints)
│   ├── Compression: gzip, brotli
│   └── Query string: Forwarded
├── Cache header: Set by backend (Cache-Control)
└── Custom domain: api.bankerchanger.com
```

---

## Blue-Green Deployment

### Overview

Two identical production environments:
- **Blue** (current): Running production traffic
- **Green** (new): Staging the new version

### Deployment Process

```
Step 1: Deploy to Green
├── Build Docker image from new code
├── Push to ECR
├── Create new ECS task definition (version +1)
├── Deploy new service with 0 traffic (0 tasks)
└── Duration: ~5 minutes

Step 2: Run Smoke Tests on Green
├── Scale green service to 1 task
├── Run automated tests:
│   ├── API health checks
│   ├── Critical user flows (login, bet placement)
│   ├── WebSocket connectivity
│   └── Database queries
├── If tests fail: Rollback immediately
└── Duration: ~3 minutes

Step 3: Gradual Traffic Shift (Canary)
├── Scale green to 1 task (10% traffic via ALB weighted target group)
├── Monitor metrics: error rate, latency, success rate
├── Compare vs Blue: Should be within 5% deviation
├── If metrics good: Continue to step 4
├── If metrics bad: Shift 100% back to Blue
└── Duration: 5-10 minutes per 10% increment

Step 4: Complete Traffic Switch
├── Shift 100% traffic to Green
├── Monitor intensively: 5-minute window
├── If all metrics healthy: Blue is now standby
└── Duration: < 1 minute

Step 5: Rollback Plan (if needed)
├── Shift 100% traffic back to Blue immediately
├── Investigate issues in Green
├── Keep Green running for debugging
└── Duration: < 30 seconds
```

### ALB Configuration for Blue-Green

```
Target Group 1 (Blue - Current):
  ├── Name: bankerchanger-blue
  ├── Port: 8080
  ├── Task count: 3
  └── Weight: 100

Target Group 2 (Green - New):
  ├── Name: bankerchanger-green
  ├── Port: 8080
  ├── Task count: 0 (starts at 0, scales during deployment)
  └── Weight: 0

Listener rule:
  ├── Condition: Path /api/*
  ├── Action: Forward to target groups
  └── Weights:
      ├── Blue: 100%
      ├── Green: 0%
      └── Updated via automation script
```

### Rollback Strategy

**Automatic Rollback Triggers**
- Error rate > 1% for >1 minute
- Latency p95 > 2 seconds for >2 minutes
- Database connection failures
- Redis connectivity loss

**Manual Rollback**
```bash
# Shift 100% traffic back to Blue immediately
aws elbv2 modify-listener --listener-arn <arn> \
  --default-actions Type=forward,ForwardConfig={...Blue:100,Green:0...}
```

### Deployment Automation

**GitOps Pipeline**
```
Git push to main →
  GitHub Actions workflow triggered
    ├── Build & test
    ├── Build Docker image
    ├── Push to ECR
    ├── Create ECS task definition
    ├── Deploy to Green
    ├── Run smoke tests
    ├── If tests pass: Start canary (10% traffic)
    ├── Monitor for 5 minutes
    ├── If metrics good: Shift to 20%, repeat
    ├── Final shift to 100%
    └── If any failure: Automatic rollback to Blue

Expected deployment time: 10-15 minutes
```

---

## Security Architecture

### Defense in Depth

**Layer 1: DDoS Protection (AWS Shield Standard)**
- Automatic DDoS mitigation for common attacks
- Layer 3/4 protection (network and transport layer)

**Layer 2: Web Application Firewall (AWS WAF)**
```
Rules:
├── OWASP Top 10 protection
│   ├── SQL injection detection
│   ├── Cross-site scripting (XSS) detection
│   ├── Cross-site request forgery (CSRF) mitigation
│   └── Local file inclusion (LFI) protection
├── Rate limiting: 100 requests per minute per IP
├── Geographic blocking: Exclude countries as needed
└── IP whitelist/blacklist: For known threats
```

**Layer 3: Network Security**
- VPC with private subnets for databases
- Security groups with least privilege rules
- NACLs for additional network filtering
- VPC Flow Logs for traffic analysis

**Layer 4: Application Security**
- HTTPS/TLS 1.3 only
- HSTS headers enforced
- Certificate pinning on mobile clients
- API authentication via JWT tokens

**Layer 5: Data Security**
- Encryption at rest: AES-256 (KMS)
- Encryption in transit: TLS 1.3
- Database: Transparent Data Encryption (TDE)
- Backups: Encrypted in S3

**Layer 6: Access Control**
- VPN-only access to management tools
- Bastion host for SSH access
- AWS IAM roles for service authentication
- AWS Secrets Manager for credentials

### Encryption Key Management

**AWS KMS Key Hierarchy**
```
Customer Master Key (CMK):
├── Created in CMK account
├── Key rotation: Automated yearly
├── Permissions: Least privilege
└── Audit: CloudTrail logs

Data Keys:
├── Generated from CMK for each encryption operation
├── Unique key per database, S3 bucket, EBS volume
├── Automatic rotation: 1 year
└── Audit trail: CloudTrail
```

---

## Monitoring & Logging Architecture

### Log Aggregation

```
ECS Task Logs (CloudWatch)
    ↓
CloudWatch Logs Groups
    ├── /ecs/bankerchanger-backend
    ├── /ecs/bankerchanger-indexer
    └── /ecs/bankerchanger-frontend
    ↓
Logstash Pipeline (ingest)
    ↓
Elasticsearch (storage)
    ↓
Kibana (visualization)
```

### Metrics Pipeline

```
Application Metrics (Prometheus format)
    ↓
Prometheus Scraper (every 30 seconds)
    ↓
Prometheus TSDB (storage)
    ↓
AlertManager (rule evaluation)
    ├── Slack notifications
    ├── PagerDuty incidents
    └── Email alerts
    ↓
Grafana (visualization)
```

### Tracing Pipeline

```
OpenTelemetry Instrumentation
    ├── Backend (Express middleware)
    ├── Indexer (manual spans)
    └── Frontend (browser API)
    ↓
OpenTelemetry Collector
    ↓
Jaeger Collector
    ↓
Elasticsearch Storage
    ↓
Jaeger Query UI
```

---

## Cost Optimization

### Resource Sizing

**ECS Task Sizing**
- Backend: 512 CPU, 1GB RAM (0.25 vCPU) - Fargate optimal
- Indexer: 512 CPU, 2GB RAM (0.25 vCPU)
- Frontend: 256 CPU, 512MB RAM (0.25 vCPU) - Minimum viable

**RDS Sizing**
- Instance: db.r6g.large (Graviton2, cost-effective)
- Multi-AZ: +50% cost but critical for HA
- Storage: gp3 with 3,000 IOPS (cost-effective SSD)

**ElastiCache Sizing**
- Instance: cache.r6g.large (3 nodes)
- Cluster mode: Enabled (sharding for distribution)

**Observability Stack**
- Jaeger: 1 collector + 3 Elasticsearch nodes
- Prometheus: 1 instance (persistent storage)
- Grafana: 1 instance (stateless)

### Cost Drivers

| Component | Monthly Cost | Optimization |
|---|---|---|
| ECS (3 tasks × 3 services) | $150 | Reserved capacity in prod |
| RDS (multi-AZ) | $600 | Graviton2 instances |
| ElastiCache (6 nodes) | $400 | Reserved capacity |
| Data transfer | $200 | CloudFront caching |
| Observability | $250 | Log retention 30 days |
| **Total** | **~$1,600** | |

### Savings Opportunities

1. **Reserved Capacity**: 33% savings with 1-year commitment (prod only)
2. **Savings Plans**: Flexible discounts across services
3. **Spot Instances**: 70% savings for non-critical workloads (dev only)
4. **Auto Scaling**: Scale down during off-hours (dev/staging)
5. **Data Transfer**: Optimize via CloudFront caching

---

## Testing Strategy

### Infrastructure Testing

**Terraform Validation**
- `terraform validate`: Syntax validation
- `terraform plan`: Plan output review
- `tflint`: Best practices and security checks
- Cost estimation: `terraform cost` plugin

**Unit Tests**
- Terraform modules tested with Terratest
- Test VPC structure, CIDR blocks, security groups
- Test RDS configuration, backup settings
- Test ElastiCache cluster mode, failover

**Integration Tests**
- Deploy infrastructure to dev environment
- Verify service connectivity
- Test database failover scenario
- Test application deployment to ECS

**Smoke Tests**
- Post-deployment checks
- Health endpoint returns 200
- Database queries work
- Redis connection works
- Logging appears in CloudWatch

### Blue-Green Deployment Testing

**Pre-Deployment Validation**
1. Code review and approval
2. Automated tests pass (unit, integration)
3. Docker image scanned for vulnerabilities
4. Staging deployment successful

**Deployment Testing**
1. Smoke tests on Green environment
2. Canary phase monitoring (10% traffic)
3. Gradual rollout to 100%
4. Rollback test (verify can quickly revert)

**Post-Deployment Monitoring**
- Metrics baseline comparison (vs Blue)
- Error rate tracking
- Latency tracking (p50, p95, p99)
- Resource utilization tracking

---

## Deployment Checklist

**Prerequisites**
- [ ] Terraform code reviewed and approved
- [ ] AWS account prepared and configured
- [ ] VPN access to bastion host tested
- [ ] DNS domain registered and configured
- [ ] SSL certificate created (ACM)

**Initial Deployment**
- [ ] VPC and networking created
- [ ] RDS database created and tested
- [ ] ElastiCache cluster created
- [ ] ECS cluster created
- [ ] ALB created and configured
- [ ] IAM roles and policies created

**Application Deployment**
- [ ] Docker images built and pushed to ECR
- [ ] ECS task definitions created
- [ ] Backend service deployed and tested
- [ ] Indexer service deployed and verified
- [ ] Frontend service deployed and tested

**Observability Setup**
- [ ] Jaeger deployed and accessible
- [ ] Elasticsearch cluster running
- [ ] Prometheus scraping metrics
- [ ] Grafana dashboards created
- [ ] AlertManager alerts configured

**Validation**
- [ ] Application health checks passing
- [ ] Logs appearing in Kibana
- [ ] Metrics appearing in Prometheus
- [ ] Traces appearing in Jaeger
- [ ] Alerts tested and working

---

## Summary

This design provides a production-grade, highly available, observable infrastructure for BANKERCHANGER capable of scaling to handle the 10k req/s target with zero-downtime deployments. The architecture prioritizes:

1. **Reliability**: Multi-AZ, multi-region ready, automatic failover
2. **Scalability**: Horizontal scaling for all stateless services
3. **Observability**: Comprehensive tracing, logging, and metrics
4. **Security**: Defense in depth with encryption, WAF, DDoS protection
5. **Operational Excellence**: IaC, blue-green deployments, disaster recovery procedures

All infrastructure is defined in Terraform for reproducibility and version control, enabling rapid deployment and recovery.
