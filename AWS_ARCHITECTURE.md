# EventXGames AWS Architecture Document

## Executive Summary

This document outlines the production-ready AWS architecture for migrating EventXGames from Azure VPS to AWS. The design prioritizes high availability, scalability, security, and cost optimization while leveraging AWS-native services.

---

## 1. Architecture Overview

```
                                    ┌─────────────────────────────────────────────────────────────┐
                                    │                        AWS Cloud                             │
┌──────────┐                        │  ┌─────────────┐     ┌──────────────────────────────────┐   │
│  Users   │───────────────────────────│  CloudFront │────▶│           S3 (Assets)            │   │
└──────────┘                        │  └──────┬──────┘     └──────────────────────────────────┘   │
                                    │         │                                                    │
                                    │         ▼                                                    │
                                    │  ┌─────────────┐     ┌──────────────────────────────────┐   │
                                    │  │     ALB     │────▶│           EKS Cluster            │   │
                                    │  └─────────────┘     │  ┌────────┐ ┌────────┐ ┌───────┐ │   │
                                    │                      │  │  API   │ │Frontend│ │ Game  │ │   │
                                    │                      │  │ Worker │ │ Worker │ │Worker │ │   │
                                    │                      │  └────┬───┘ └────────┘ └───┬───┘ │   │
                                    │                      │       │      ┌────────┐    │     │   │
                                    │                      │       │      │ Asset  │    │     │   │
                                    │                      │       │      │ Worker │    │     │   │
                                    │                      │       │      └────────┘    │     │   │
                                    │                      │       │      ┌────────┐    │     │   │
                                    │                      │       │      │  Chat  │    │     │   │
                                    │                      │       │      │ Worker │    │     │   │
                                    │                      │       │      └────────┘    │     │   │
                                    │                      └───────┼──────────────────┼─────┘    │
                                    │                              │                  │          │
                                    │                              ▼                  ▼          │
                                    │  ┌──────────────────────────────────────────────────────┐  │
                                    │  │              Aurora PostgreSQL (Multi-AZ)            │  │
                                    │  └──────────────────────────────────────────────────────┘  │
                                    │                              │                              │
                                    │                              ▼                              │
                                    │  ┌──────────────────────────────────────────────────────┐  │
                                    │  │              ElastiCache Redis (Cluster Mode)        │  │
                                    │  └──────────────────────────────────────────────────────┘  │
                                    │                              │                              │
                                    │                              ▼                              │
                                    │  ┌──────────────────────────────────────────────────────┐  │
                                    │  │                    Amazon Bedrock                    │  │
                                    │  └──────────────────────────────────────────────────────┘  │
                                    └─────────────────────────────────────────────────────────────┘
```

---

## 2. EKS Cluster Design

### 2.1 Cluster Configuration

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Kubernetes Version** | 1.29 | Latest stable with long-term support |
| **Region** | us-east-1 (Primary), us-west-2 (DR) | Low latency, full service availability |
| **Availability Zones** | 3 AZs minimum | High availability |
| **Node Groups** | Managed Node Groups | Simplified operations, automatic updates |
| **CNI** | AWS VPC CNI | Native VPC networking, security groups per pod |
| **Service Mesh** | AWS App Mesh (optional) | Observability, traffic management |

### 2.2 Node Group Configuration

#### System Node Group
```yaml
name: system-nodes
instanceTypes: [m6i.large]
minSize: 2
maxSize: 4
desiredSize: 3
labels:
  role: system
taints:
  - key: CriticalAddonsOnly
    effect: NoSchedule
```

#### Application Node Group
```yaml
name: app-nodes
instanceTypes: [m6i.xlarge, m6i.2xlarge]
minSize: 3
maxSize: 20
desiredSize: 6
labels:
  role: application
capacityType: ON_DEMAND  # Production workloads
```

#### Spot Node Group (Non-Critical Workloads)
```yaml
name: spot-nodes
instanceTypes: [m6i.xlarge, m6i.2xlarge, m5.xlarge, m5.2xlarge]
minSize: 0
maxSize: 10
desiredSize: 2
labels:
  role: spot-workloads
capacityType: SPOT
taints:
  - key: spot-instance
    effect: NoSchedule
```

### 2.3 Workload Specifications

#### API Worker
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-worker
  namespace: eventxgames
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-worker
  template:
    spec:
      containers:
      - name: api
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-worker
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

#### Frontend Worker
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-worker
  namespace: eventxgames
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: frontend
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-worker-hpa
spec:
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

#### Game Worker
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-worker
  namespace: eventxgames
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: game
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "4000m"
            memory: "8Gi"
        env:
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: game-worker-hpa
spec:
  minReplicas: 5
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
  - type: Pods
    pods:
      metric:
        name: active_game_sessions
      target:
        type: AverageValue
        averageValue: "100"
```

#### Asset Worker
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: asset-worker
  namespace: eventxgames
spec:
  replicas: 2
  template:
    spec:
      tolerations:
      - key: spot-instance
        operator: Exists
        effect: NoSchedule
      nodeSelector:
        role: spot-workloads
      containers:
      - name: asset
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: asset-worker-hpa
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

#### Chat Worker
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chat-worker
  namespace: eventxgames
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: chat
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        ports:
        - containerPort: 8080  # HTTP
        - containerPort: 8081  # WebSocket
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chat-worker-hpa
spec:
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 75
  - type: Pods
    pods:
      metric:
        name: websocket_connections
      target:
        type: AverageValue
        averageValue: "1000"
```

### 2.4 EKS Add-ons

| Add-on | Version | Purpose |
|--------|---------|---------|
| AWS Load Balancer Controller | 2.7+ | ALB/NLB ingress |
| CoreDNS | Latest | DNS resolution |
| kube-proxy | Match EKS | Network proxy |
| Amazon VPC CNI | Latest | Pod networking |
| AWS EBS CSI Driver | Latest | Persistent volumes |
| Metrics Server | Latest | HPA metrics |
| Cluster Autoscaler | Latest | Node scaling |
| Fluent Bit | Latest | Log forwarding to CloudWatch |

---

## 3. Aurora PostgreSQL Setup

### 3.1 Cluster Configuration

```hcl
# Terraform Configuration
resource "aws_rds_cluster" "eventxgames" {
  cluster_identifier      = "eventxgames-aurora-cluster"
  engine                  = "aurora-postgresql"
  engine_version          = "15.4"
  engine_mode             = "provisioned"
  database_name           = "eventxgames"
  master_username         = "admin"
  master_password         = var.db_password  # Use Secrets Manager
  
  # Multi-AZ
  availability_zones      = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  # Networking
  db_subnet_group_name    = aws_db_subnet_group.private.name
  vpc_security_group_ids  = [aws_security_group.aurora.id]
  
  # Storage
  storage_encrypted       = true
  kms_key_id             = aws_kms_key.rds.arn
  
  # Backup
  backup_retention_period = 35
  preferred_backup_window = "03:00-04:00"
  
  # Maintenance
  preferred_maintenance_window = "sun:04:00-sun:05:00"
  
  # Performance Insights
  performance_insights_enabled    = true
  performance_insights_retention_period = 7
  
  # Deletion protection
  deletion_protection     = true
  skip_final_snapshot     = false
  final_snapshot_identifier = "eventxgames-final-snapshot"
  
  # Serverless v2 scaling (optional)
  serverlessv2_scaling_configuration {
    min_capacity = 2
    max_capacity = 64
  }
}

# Writer Instance
resource "aws_rds_cluster_instance" "writer" {
  identifier           = "eventxgames-writer"
  cluster_identifier   = aws_rds_cluster.eventxgames.id
  instance_class       = "db.r6g.2xlarge"
  engine               = aws_rds_cluster.eventxgames.engine
  engine_version       = aws_rds_cluster.eventxgames.engine_version
  
  performance_insights_enabled = true
  monitoring_interval  = 60
  monitoring_role_arn  = aws_iam_role.rds_monitoring.arn
}

# Reader Instances (2 replicas)
resource "aws_rds_cluster_instance" "readers" {
  count                = 2
  identifier           = "eventxgames-reader-${count.index + 1}"
  cluster_identifier   = aws_rds_cluster.eventxgames.id
  instance_class       = "db.r6g.xlarge"
  engine               = aws_rds_cluster.eventxgames.engine
  engine_version       = aws_rds_cluster.eventxgames.engine_version
  
  performance_insights_enabled = true
}
```

### 3.2 Parameter Groups

```hcl
resource "aws_rds_cluster_parameter_group" "eventxgames" {
  family      = "aurora-postgresql15"
  name        = "eventxgames-cluster-params"
  description = "EventXGames Aurora PostgreSQL cluster parameters"

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements,auto_explain"
  }
  
  parameter {
    name  = "log_statement"
    value = "ddl"
  }
  
  parameter {
    name  = "log_min_duration_statement"
    value = "1000"  # Log queries > 1 second
  }
  
  parameter {
    name  = "max_connections"
    value = "LEAST({DBInstanceClassMemory/9531392},5000)"
  }
  
  parameter {
    name  = "work_mem"
    value = "262144"  # 256MB
  }
}
```

### 3.3 Connection Pooling (PgBouncer on EKS)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: eventxgames
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: pgbouncer
        image: edoburu/pgbouncer:1.21.0
        ports:
        - containerPort: 5432
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: aurora-credentials
              key: writer-url
        - name: POOL_MODE
          value: "transaction"
        - name: MAX_CLIENT_CONN
          value: "1000"
        - name: DEFAULT_POOL_SIZE
          value: "50"
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

### 3.4 Estimated Sizing

| Component | Specification | Monthly Cost (Est.) |
|-----------|--------------|---------------------|
| Writer | db.r6g.2xlarge (8 vCPU, 64GB) | ~$1,200 |
| Reader 1 | db.r6g.xlarge (4 vCPU, 32GB) | ~$600 |
| Reader 2 | db.r6g.xlarge (4 vCPU, 32GB) | ~$600 |
| Storage | 500GB + IOPS | ~$150 |
| Backups | 35-day retention | ~$50 |
| **Total** | | **~$2,600/month** |

---

## 4. ElastiCache Redis Configuration

### 4.1 Cluster Configuration

```hcl
resource "aws_elasticache_replication_group" "eventxgames" {
  replication_group_id       = "eventxgames-redis"
  description                = "EventXGames Redis cluster"
  
  # Engine
  engine                     = "redis"
  engine_version             = "7.1"
  node_type                  = "cache.r6g.xlarge"
  
  # Cluster Mode Enabled
  num_node_groups            = 3
  replicas_per_node_group    = 2
  
  # Networking
  subnet_group_name          = aws_elasticache_subnet_group.private.name
  security_group_ids         = [aws_security_group.redis.id]
  port                       = 6379
  
  # Security
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  kms_key_id                 = aws_kms_key.elasticache.arn
  auth_token                 = var.redis_auth_token  # Use Secrets Manager
  
  # High Availability
  automatic_failover_enabled = true
  multi_az_enabled           = true
  
  # Maintenance
  maintenance_window         = "sun:05:00-sun:06:00"
  snapshot_retention_limit   = 7
  snapshot_window            = "04:00-05:00"
  
  # Parameters
  parameter_group_name       = aws_elasticache_parameter_group.eventxgames.name
  
  # Auto minor version upgrade
  auto_minor_version_upgrade = true
}

resource "aws_elasticache_parameter_group" "eventxgames" {
  family = "redis7"
  name   = "eventxgames-redis-params"
  
  parameter {
    name  = "maxmemory-policy"
    value = "volatile-lru"
  }
  
  parameter {
    name  = "notify-keyspace-events"
    value = "Ex"  # Enable keyspace events for expiration
  }
  
  parameter {
    name  = "cluster-enabled"
    value = "yes"
  }
}
```

### 4.2 Use Cases & Key Prefixes

| Use Case | Key Prefix | TTL | Purpose |
|----------|------------|-----|---------|
| Session Storage | `session:` | 24h | User sessions |
| Game State | `game:` | 1h | Active game sessions |
| Leaderboards | `lb:` | 5m | Real-time leaderboards (sorted sets) |
| Rate Limiting | `rate:` | 1m | API rate limiting |
| Cache | `cache:` | Varies | General caching |
| Pub/Sub | `pubsub:` | N/A | Real-time messaging |
| Distributed Locks | `lock:` | 30s | Distributed locking |

### 4.3 Connection Configuration (Application)

```yaml
# Kubernetes ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: eventxgames
data:
  REDIS_CLUSTER_MODE: "enabled"
  REDIS_CLUSTER_ENDPOINTS: "eventxgames-redis.xxxxx.clustercfg.use1.cache.amazonaws.com:6379"
  REDIS_TLS_ENABLED: "true"
  REDIS_MAX_CONNECTIONS: "100"
  REDIS_CONNECTION_TIMEOUT: "5000"
  REDIS_READ_TIMEOUT: "3000"
```

### 4.4 Estimated Sizing

| Component | Specification | Monthly Cost (Est.) |
|-----------|--------------|---------------------|
| 3 Shards | cache.r6g.xlarge (4 vCPU, 26GB) | ~$1,350 |
| 6 Replicas | cache.r6g.xlarge × 6 | ~$2,700 |
| Data Transfer | Estimated | ~$100 |
| **Total** | | **~$4,150/month** |

---

## 5. S3 + CloudFront for Assets

### 5.1 S3 Bucket Configuration

```hcl
# Primary Assets Bucket
resource "aws_s3_bucket" "assets" {
  bucket = "eventxgames-assets-${var.environment}"
}

resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id
  
  rule {
    id     = "transition-to-ia"
    status = "Enabled"
    
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 180
      storage_class = "GLACIER_IR"
    }
    
    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }
    
    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}

resource "aws_s3_bucket_cors_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "HEAD"]
    allowed_origins = ["https://eventxgames.com", "https://*.eventxgames.com"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3600
  }
}

# Replication to DR region
resource "aws_s3_bucket_replication_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-to-dr"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.assets_dr.arn
      storage_class = "STANDARD"
      
      encryption_configuration {
        replica_kms_key_id = aws_kms_key.s3_dr.arn
      }
    }

    source_selection_criteria {
      sse_kms_encrypted_objects {
        status = "Enabled"
      }
    }
  }
}
```

### 5.2 CloudFront Distribution

```hcl
resource "aws_cloudfront_distribution" "assets" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "EventXGames Asset CDN"
  default_root_object = "index.html"
  price_class         = "PriceClass_All"
  
  aliases = ["cdn.eventxgames.com"]
  
  origin {
    domain_name              = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id                = "S3-Assets"
    origin_access_control_id = aws_cloudfront_origin_access_control.assets.id
    
    origin_shield {
      enabled              = true
      origin_shield_region = "us-east-1"
    }
  }
  
  # API Origin (for dynamic content)
  origin {
    domain_name = aws_lb.api.dns_name
    origin_id   = "API-ALB"
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }
  
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-Assets"
    
    forwarded_values {
      query_string = false
      headers      = ["Origin", "Access-Control-Request-Method", "Access-Control-Request-Headers"]
      
      cookies {
        forward = "none"
      }
    }
    
    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 86400    # 1 day
    max_ttl                = 31536000 # 1 year
    compress               = true
    
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security.id
  }
  
  # Cache behavior for game assets (long TTL)
  ordered_cache_behavior {
    path_pattern     = "/games/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-Assets"
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    min_ttl                = 604800    # 7 days
    default_ttl            = 2592000   # 30 days
    max_ttl                = 31536000  # 1 year
    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }
  
  # API passthrough behavior
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "API-ALB"
    
    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Host", "Origin"]
      
      cookies {
        forward = "all"
      }
    }
    
    min_ttl                = 0
    default_ttl            = 0
    max_ttl                = 0
    compress               = true
    viewer_protocol_policy = "https-only"
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cdn.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
  
  logging_config {
    include_cookies = false
    bucket          = aws_s3_bucket.logs.bucket_domain_name
    prefix          = "cloudfront/"
  }
  
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn
}

# Origin Access Control
resource "aws_cloudfront_origin_access_control" "assets" {
  name                              = "eventxgames-assets-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# Security Headers
resource "aws_cloudfront_response_headers_policy" "security" {
  name = "eventxgames-security-headers"
  
  security_headers_config {
    content_type_options {
      override = true
    }
    
    frame_options {
      frame_option = "DENY"
      override     = true
    }
    
    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }
    
    strict_transport_security {
      access_control_max_age_sec = 31536000
      include_subdomains         = true
      preload                    = true
      override                   = true
    }
    
    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }
  }
  
  cors_config {
    access_control_allow_credentials = false
    
    access_control_allow_headers {
      items = ["*"]
    }
    
    access_control_allow_methods {
      items = ["GET", "HEAD", "OPTIONS"]
    }
    
    access_control_allow_origins {
      items = ["https://eventxgames.com", "https://*.eventxgames.com"]
    }
    
    origin_override = true
  }
}
```

### 5.3 Estimated Costs

| Component | Specification | Monthly Cost (Est.) |
|-----------|--------------|---------------------|
| S3 Storage | 1TB Standard | ~$23 |
| S3 Requests | 10M GET, 1M PUT | ~$5 |
| CloudFront | 10TB transfer out | ~$850 |
| CloudFront Requests | 100M requests | ~$75 |
| Origin Shield | Enabled | ~$50 |
| **Total** | | **~$1,000/month** |

---

## 6. Amazon Bedrock Integration

### 6.1 Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Bedrock Integration                                │
│                                                                              │
│  ┌──────────┐     ┌─────────────┐     ┌───────────────┐     ┌──────────┐   │
│  │   API    │────▶│  Bedrock    │────▶│   Claude 3    │     │ Knowledge │   │
│  │  Worker  │     │  Runtime    │     │   Sonnet/     │◀────│   Bases   │   │
│  └──────────┘     └─────────────┘     │   Haiku       │     └──────────┘   │
│       │                               └───────────────┘                     │
│       │           ┌─────────────┐     ┌───────────────┐                     │
│       └──────────▶│  Guardrails │────▶│   Content     │                     │
│                   │             │     │   Filtering   │                     │
│                   └─────────────┘     └───────────────┘                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 IAM Configuration

```hcl
# Bedrock Access Role for EKS
resource "aws_iam_role" "bedrock_access" {
  name = "eventxgames-bedrock-access"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:eventxgames:bedrock-sa"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "bedrock_access" {
  name = "bedrock-access-policy"
  role = aws_iam_role.bedrock_access.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "bedrock:InvokeModel",
          "bedrock:InvokeModelWithResponseStream"
        ]
        Resource = [
          "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-*",
          "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-haiku-*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "bedrock:ApplyGuardrail"
        ]
        Resource = aws_bedrock_guardrail.eventxgames.arn
      },
      {
        Effect = "Allow"
        Action = [
          "bedrock:Retrieve"
        ]
        Resource = aws_bedrock_knowledge_base.game_docs.arn
      }
    ]
  })
}

# Kubernetes Service Account
resource "kubernetes_service_account" "bedrock" {
  metadata {
    name      = "bedrock-sa"
    namespace = "eventxgames"
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.bedrock_access.arn
    }
  }
}
```

### 6.3 Guardrails Configuration

```hcl
resource "aws_bedrock_guardrail" "eventxgames" {
  name        = "eventxgames-guardrail"
  description = "Content filtering for EventXGames AI features"
  
  blocked_input_messaging  = "I cannot process this request due to content policy."
  blocked_outputs_messaging = "I cannot provide this response due to content policy."
  
  content_policy_config {
    filters_config {
      input_strength  = "MEDIUM"
      output_strength = "MEDIUM"
      type           = "HATE"
    }
    
    filters_config {
      input_strength  = "HIGH"
      output_strength = "HIGH"
      type           = "VIOLENCE"
    }
    
    filters_config {
      input_strength  = "HIGH"
      output_strength = "HIGH"
      type           = "SEXUAL"
    }
    
    filters_config {
      input_strength  = "MEDIUM"
      output_strength = "MEDIUM"
      type           = "INSULTS"
    }
  }
  
  word_policy_config {
    managed_word_lists_config {
      type = "PROFANITY"
    }
  }
  
  sensitive_information_policy_config {
    pii_entities_config {
      action = "ANONYMIZE"
      type   = "EMAIL"
    }
    
    pii_entities_config {
      action = "ANONYMIZE"
      type   = "PHONE"
    }
    
    pii_entities_config {
      action = "BLOCK"
      type   = "CREDIT_DEBIT_CARD_NUMBER"
    }
  }
}
```

### 6.4 Knowledge Base for Game Documentation

```hcl
resource "aws_bedrock_knowledge_base" "game_docs" {
  name     = "eventxgames-knowledge-base"
  role_arn = aws_iam_role.bedrock_kb.arn
  
  knowledge_base_configuration {
    type = "VECTOR"
    
    vector_knowledge_base_configuration {
      embedding_model_arn = "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v1"
    }
  }
  
  storage_configuration {
    type = "OPENSEARCH_SERVERLESS"
    
    opensearch_serverless_configuration {
      collection_arn = aws_opensearchserverless_collection.kb.arn
      vector_index_name = "game-docs-index"
      
      field_mapping {
        metadata_field = "metadata"
        text_field     = "text"
        vector_field   = "vector"
      }
    }
  }
}
```

### 6.5 Application Integration Example

```python
# Python SDK usage example
import boto3
from botocore.config import Config

# Configure client with retry and timeout
bedrock_config = Config(
    region_name='us-east-1',
    retries={'max_attempts': 3, 'mode': 'adaptive'},
    read_timeout=60
)

bedrock_runtime = boto3.client('bedrock-runtime', config=bedrock_config)

def generate_game_hint(user_context: str, game_state: dict) -> str:
    """Generate contextual game hints using Claude."""
    
    response = bedrock_runtime.invoke_model(
        modelId='anthropic.claude-3-haiku-20240307-v1:0',
        contentType='application/json',
        accept='application/json',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'max_tokens': 200,
            'messages': [
                {
                    'role': 'user',
                    'content': f"Game context: {game_state}\nUser query: {user_context}\nProvide a brief, helpful hint."
                }
            ],
            'temperature': 0.7
        }),
        guardrailIdentifier='eventxgames-guardrail',
        guardrailVersion='DRAFT'
    )
    
    result = json.loads(response['body'].read())
    return result['content'][0]['text']
```

### 6.6 Estimated Costs

| Component | Usage Estimate | Monthly Cost (Est.) |
|-----------|---------------|---------------------|
| Claude 3 Haiku | 10M input tokens, 5M output tokens | ~$300 |
| Claude 3 Sonnet | 1M input tokens, 500K output tokens | ~$200 |
| Guardrails | 10M assessments | ~$100 |
| Knowledge Base | Retrieval queries | ~$50 |
| **Total** | | **~$650/month** |

---

## 7. Multi-Region Failover Strategy

### 7.1 Architecture Overview

```
                           ┌─────────────────────────────────────┐
                           │          Route 53                   │
                           │   (Health-Based Routing)            │
                           └──────────┬──────────┬───────────────┘
                                      │          │
                    ┌─────────────────┴──┐  ┌────┴─────────────────┐
                    ▼                    │  │                      ▼
           ┌────────────────┐           │  │           ┌────────────────┐
           │  us-east-1     │           │  │           │  us-west-2     │
           │  (PRIMARY)     │           │  │           │  (DR)          │
           ├────────────────┤           │  │           ├────────────────┤
           │ • EKS Cluster  │           │  │           │ • EKS Cluster  │
           │ • Aurora (W/R) │◀──────────┴──┴──────────▶│ • Aurora (R)   │
           │ • ElastiCache  │    Global Database      │ • ElastiCache  │
           │ • CloudFront   │    Cross-Region         │                │
           └────────────────┘    Replication          └────────────────┘
```

### 7.2 Route 53 Health Checks & Failover

```hcl
# Primary Health Check
resource "aws_route53_health_check" "primary" {
  fqdn              = aws_lb.api_primary.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  tags = {
    Name = "eventxgames-primary-health"
  }
}

# Secondary Health Check
resource "aws_route53_health_check" "secondary" {
  fqdn              = aws_lb.api_secondary.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  tags = {
    Name = "eventxgames-secondary-health"
  }
}

# Primary Record (Failover Routing)
resource "aws_route53_record" "api_primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.eventxgames.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = aws_lb.api_primary.dns_name
    zone_id                = aws_lb.api_primary.zone_id
    evaluate_target_health = true
  }
}

# Secondary Record (Failover Routing)
resource "aws_route53_record" "api_secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.eventxgames.com"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier  = "secondary"
  health_check_id = aws_route53_health_check.secondary.id

  alias {
    name                   = aws_lb.api_secondary.dns_name
    zone_id                = aws_lb.api_secondary.zone_id
    evaluate_target_health = true
  }
}
```

### 7.3 Aurora Global Database

```hcl
# Global Cluster
resource "aws_rds_global_cluster" "eventxgames" {
  global_cluster_identifier = "eventxgames-global"
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  database_name             = "eventxgames"
  storage_encrypted         = true
}

# Primary Cluster (us-east-1)
resource "aws_rds_cluster" "primary" {
  provider                  = aws.us_east_1
  cluster_identifier        = "eventxgames-primary"
  global_cluster_identifier = aws_rds_global_cluster.eventxgames.id
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  database_name             = "eventxgames"
  master_username           = "admin"
  master_password           = var.db_password
  
  # Enable write forwarding for DR reads
  enable_global_write_forwarding = false
}

# Secondary Cluster (us-west-2)
resource "aws_rds_cluster" "secondary" {
  provider                  = aws.us_west_2
  cluster_identifier        = "eventxgames-secondary"
  global_cluster_identifier = aws_rds_global_cluster.eventxgames.id
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  
  # This is a read replica, no master credentials needed
  skip_final_snapshot       = true
  
  depends_on = [aws_rds_cluster.primary]
}
```

### 7.4 ElastiCache Global Datastore

```hcl
resource "aws_elasticache_global_replication_group" "eventxgames" {
  global_replication_group_id_suffix = "eventxgames"
  primary_replication_group_id       = aws_elasticache_replication_group.primary.id
  
  global_replication_group_description = "EventXGames Global Redis"
}

# Secondary Region Replication Group
resource "aws_elasticache_replication_group" "secondary" {
  provider                      = aws.us_west_2
  replication_group_id          = "eventxgames-redis-secondary"
  description                   = "EventXGames Redis DR"
  global_replication_group_id   = aws_elasticache_global_replication_group.eventxgames.global_replication_group_id
  
  num_cache_clusters            = 2
  node_type                     = "cache.r6g.large"
  
  automatic_failover_enabled    = true
  multi_az_enabled              = true
}
```

### 7.5 Failover Procedures

#### Automated Failover (Route 53)
- Health checks monitor primary region every 10 seconds
- After 3 consecutive failures (~30 seconds), Route 53 fails over to secondary
- DNS TTL: 60 seconds (total failover time: ~90 seconds)

#### Aurora Failover (Manual Promotion)
```bash
# Promote secondary to primary (run during DR event)
aws rds failover-global-cluster \
    --global-cluster-identifier eventxgames-global \
    --target-db-cluster-identifier arn:aws:rds:us-west-2:ACCOUNT:cluster:eventxgames-secondary
```

#### Redis Failover (Automatic within region, manual cross-region)
```bash
# Promote secondary global datastore region
aws elasticache failover-global-replication-group \
    --global-replication-group-id eventxgames \
    --primary-region us-west-2 \
    --primary-replication-group-id eventxgames-redis-secondary
```

### 7.6 RTO/RPO Targets

| Component | RTO | RPO | Notes |
|-----------|-----|-----|-------|
| DNS Failover | ~90 seconds | 0 | Automatic via Route 53 |
| Aurora Global | ~1 minute | <1 second | Synchronous replication to ~1s async |
| ElastiCache | ~2 minutes | <1 second | Global Datastore replication |
| S3 Assets | 0 (Always available) | ~15 minutes | Cross-region replication |
| Overall | **<5 minutes** | **<1 minute** | Manual intervention may be required |

### 7.7 DR Cost Estimate (Secondary Region)

| Component | Specification | Monthly Cost (Est.) |
|-----------|--------------|---------------------|
| EKS Cluster (standby) | Minimal nodes | ~$200 |
| Aurora Reader | db.r6g.large × 2 | ~$400 |
| ElastiCache | cache.r6g.large × 2 | ~$600 |
| NAT Gateway | Single AZ | ~$100 |
| Data Transfer | Cross-region | ~$200 |
| **Total DR Overhead** | | **~$1,500/month** |

---

## 8. Cost Optimization Recommendations

### 8.1 Compute Optimization

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| **Spot Instances for non-critical workloads** | 60-70% | Asset Workers, batch jobs |
| **Reserved Instances (1-year)** | 30-40% | Production nodes, databases |
| **Savings Plans (Compute)** | 20-30% | Flexible compute commitment |
| **Graviton Instances** | 20% | r6g, m6g instead of Intel |
| **Right-sizing** | 15-25% | Monthly instance optimization |

### 8.2 Database Optimization

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| **Aurora I/O-Optimized** | 25% on I/O heavy | If I/O > 25% of costs |
| **Reserved Instances** | 30-40% | 1-year term for production |
| **Read Replicas for read-heavy** | Reduced primary load | Route reads appropriately |

### 8.3 Storage & Transfer Optimization

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| **S3 Intelligent Tiering** | 20-40% | Automatic lifecycle |
| **CloudFront caching** | 40-60% origin costs | Maximize cache hit ratio |
| **VPC Endpoints** | Eliminate NAT costs | S3, DynamoDB, ECR endpoints |
| **Compression** | 30-50% transfer | gzip/brotli for text assets |

### 8.4 AI/ML Optimization

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| **Haiku for simple tasks** | 75% vs Sonnet | Route by complexity |
| **Prompt caching** | 90% for repeated prefixes | Cache system prompts |
| **Batch inference** | 50% | Non-real-time processing |
| **Response length limits** | Proportional | Set appropriate max_tokens |

### 8.5 Monthly Cost Summary

| Component | On-Demand | Optimized | Savings |
|-----------|-----------|-----------|---------|
| EKS/EC2 | $8,000 | $5,500 | 31% |
| Aurora PostgreSQL | $2,600 | $1,800 | 31% |
| ElastiCache Redis | $4,150 | $2,900 | 30% |
| S3 + CloudFront | $1,000 | $800 | 20% |
| Bedrock | $650 | $400 | 38% |
| DR Region | $1,500 | $1,200 | 20% |
| Networking/Other | $1,500 | $1,200 | 20% |
| **Total Monthly** | **$19,400** | **$13,800** | **29%** |
| **Annual** | **$232,800** | **$165,600** | **$67,200 saved** |

### 8.6 Cost Monitoring & Governance

```hcl
# AWS Budgets Alert
resource "aws_budgets_budget" "monthly" {
  name         = "eventxgames-monthly-budget"
  budget_type  = "COST"
  limit_amount = "15000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["platform-team@eventxgames.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["platform-team@eventxgames.com", "finance@eventxgames.com"]
  }
}

# Cost Allocation Tags
locals {
  required_tags = {
    Project     = "EventXGames"
    Environment = var.environment
    CostCenter  = "Platform"
    ManagedBy   = "Terraform"
  }
}
```

---

## 9. Security Architecture

### 9.1 Network Security

```hcl
# VPC Configuration
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0"

  name = "eventxgames-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  intra_subnets   = ["10.0.201.0/24", "10.0.202.0/24", "10.0.203.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false
  one_nat_gateway_per_az = true

  enable_dns_hostnames = true
  enable_dns_support   = true

  # VPC Flow Logs
  enable_flow_log                      = true
  create_flow_log_cloudwatch_log_group = true
  create_flow_log_cloudwatch_iam_role  = true
}
```

### 9.2 WAF Configuration

```hcl
resource "aws_wafv2_web_acl" "cloudfront" {
  name        = "eventxgames-waf"
  description = "WAF for EventXGames"
  scope       = "CLOUDFRONT"
  provider    = aws.us_east_1

  default_action {
    allow {}
  }

  # AWS Managed Rules
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "KnownBadInputs"
      sampled_requests_enabled   = true
    }
  }

  # Rate limiting
  rule {
    name     = "RateLimitRule"
    priority = 3

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "eventxgames-waf"
    sampled_requests_enabled   = true
  }
}
```

### 9.3 Secrets Management

```hcl
# Database credentials
resource "aws_secretsmanager_secret" "db_credentials" {
  name        = "eventxgames/database/credentials"
  description = "Aurora PostgreSQL credentials"
  
  replica {
    region = "us-west-2"  # DR region
  }
}

# External Secrets Operator for Kubernetes
resource "helm_release" "external_secrets" {
  name       = "external-secrets"
  repository = "https://charts.external-secrets.io"
  chart      = "external-secrets"
  namespace  = "external-secrets"
  version    = "0.9.0"
}
```

---

## 10. Observability Stack

### 10.1 CloudWatch Configuration

```hcl
# Container Insights
resource "aws_cloudwatch_log_group" "eks" {
  name              = "/aws/eks/eventxgames/containers"
  retention_in_days = 30
}

# Custom Dashboard
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "EventXGames-Overview"
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "EKS CPU Utilization"
          region = "us-east-1"
          metrics = [
            ["AWS/EKS", "pod_cpu_utilization", "ClusterName", "eventxgames"]
          ]
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "Aurora Connections"
          region = "us-east-1"
          metrics = [
            ["AWS/RDS", "DatabaseConnections", "DBClusterIdentifier", "eventxgames-aurora-cluster"]
          ]
        }
      }
    ]
  })
}
```

### 10.2 Alarms

```hcl
# High CPU Alarm
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "eventxgames-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 60
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "CPU utilization exceeds 80%"
  
  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.eks_nodes.name
  }
}

# Database Connection Alarm
resource "aws_cloudwatch_metric_alarm" "db_connections" {
  alarm_name          = "eventxgames-db-connections"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = 60
  statistic           = "Average"
  threshold           = 800
  alarm_description   = "Database connections approaching limit"
  
  alarm_actions = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    DBClusterIdentifier = aws_rds_cluster.eventxgames.cluster_identifier
  }
}
```

---

## 11. Implementation Roadmap

### Phase 1: Foundation (Week 1-2)
- [ ] Set up AWS accounts and organizations
- [ ] Deploy VPC and networking infrastructure
- [ ] Configure IAM roles and policies
- [ ] Set up Terraform state management (S3 + DynamoDB)
- [ ] Deploy EKS cluster (without workloads)

### Phase 2: Data Layer (Week 3-4)
- [ ] Deploy Aurora PostgreSQL cluster
- [ ] Deploy ElastiCache Redis cluster
- [ ] Configure S3 buckets and replication
- [ ] Set up secrets management
- [ ] Test database connectivity from EKS

### Phase 3: Application Deployment (Week 5-6)
- [ ] Deploy application workloads to EKS
- [ ] Configure ALB and ingress
- [ ] Set up CloudFront distribution
- [ ] Configure Bedrock integration
- [ ] End-to-end testing

### Phase 4: DR & Production Readiness (Week 7-8)
- [ ] Deploy DR region infrastructure
- [ ] Configure Aurora Global Database
- [ ] Set up Route 53 failover
- [ ] DR failover testing
- [ ] Load testing and performance tuning

### Phase 5: Go-Live (Week 9)
- [ ] Final security review
- [ ] DNS cutover
- [ ] Monitor and optimize
- [ ] Runbook documentation

---

## 12. Appendix

### A. Terraform Module Structure

```
terraform/
├── environments/
│   ├── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── staging/
├── modules/
│   ├── eks/
│   ├── aurora/
│   ├── elasticache/
│   ├── cloudfront/
│   └── networking/
└── global/
    ├── iam/
    └── route53/
```

### B. Key AWS Service Limits to Request

| Service | Default Limit | Recommended |
|---------|--------------|-------------|
| VPCs per Region | 5 | 10 |
| EC2 vCPU (On-Demand) | 5 | 200 |
| EC2 vCPU (Spot) | 5 | 100 |
| EKS Clusters | 100 | Sufficient |
| RDS Instances | 40 | 60 |
| ElastiCache Nodes | 300 | Sufficient |

### C. Contact & Support

- **AWS Solutions Architect**: Available for architecture review
- **AWS Support**: Business or Enterprise support recommended
- **Terraform Registry**: Official AWS modules

---

*Document Version: 1.0*  
*Created: 2026-04-16*  
*Author: AWS Solutions Architect Agent*
