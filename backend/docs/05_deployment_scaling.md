# Envoy Fleet Management System - Deployment & Scaling Strategy

## 1. Overview

This document outlines the deployment architecture, scaling strategies, and operational procedures for the Envoy Fleet Management System built with Rust, Axum, and SQLx.

## 2. Development Environment

### 2.1 Local Development Setup

```bash
# Prerequisites
- Rust 1.75+
- PostgreSQL 14+
- Redis 7+
- Docker 24+
- Docker Compose 2.20+

# Setup steps
git clone https://github.com/org/envoy-control-plane.git
cd envoy-control-plane/backend

# Install SQLx CLI
cargo install sqlx-cli --features postgres

# Setup environment variables
cp .env.example .env
# Edit .env with your local configuration

# Start infrastructure services
docker-compose -f docker-compose.dev.yml up -d

# Run database migrations
sqlx migrate run

# Build and run the application
cargo build --release
cargo run

# Run tests
cargo test
cargo test --integration
```

### 2.2 Docker Compose Development

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: envoy_control
      POSTGRES_USER: envoy_user
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    networks:
      - envoy-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - envoy-network

  nats:
    image: nats-streaming:latest
    ports:
      - "4222:4222"
      - "8222:8222"
    command: ["-st", "file", "-dir", "/data"]
    volumes:
      - nats_data:/data
    networks:
      - envoy-network

  api:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgres://envoy_user:dev_password@postgres:5432/envoy_control
      REDIS_URL: redis://redis:6379
      NATS_URL: nats://nats:4222
      RUST_LOG: info
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
      - nats
    networks:
      - envoy-network
    volumes:
      - ./src:/app/src
      - ./Cargo.toml:/app/Cargo.toml
      - ./Cargo.lock:/app/Cargo.lock

volumes:
  postgres_data:
  redis_data:
  nats_data:

networks:
  envoy-network:
    driver: bridge
```

## 3. Production Deployment

### 3.1 Multi-Stage Dockerfile

```dockerfile
# Dockerfile
# Build stage
FROM rust:1.75-alpine AS builder

RUN apk add --no-cache musl-dev openssl-dev pkgconfig

WORKDIR /app

# Copy dependency files
COPY Cargo.toml Cargo.lock ./

# Build dependencies (cached layer)
RUN mkdir src && \
    echo "fn main() {}" > src/main.rs && \
    cargo build --release && \
    rm -rf src

# Copy source code
COPY src ./src
COPY migrations ./migrations
COPY sqlx-data.json ./

# Build application
RUN cargo build --release

# Runtime stage
FROM alpine:3.19

RUN apk add --no-cache openssl ca-certificates

WORKDIR /app

# Copy binary from builder
COPY --from=builder /app/target/release/envoy-control-plane /app/
COPY --from=builder /app/migrations /app/migrations

# Create non-root user
RUN addgroup -g 1000 envoy && \
    adduser -D -u 1000 -G envoy envoy

USER envoy

EXPOSE 8080

ENTRYPOINT ["/app/envoy-control-plane"]
```

### 3.2 Docker Compose Production

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  postgres-primary:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: envoy_control
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_REPLICATION_MODE: master
      POSTGRES_REPLICATION_USER: replicator
      POSTGRES_REPLICATION_PASSWORD: ${REPLICATION_PASSWORD}
    volumes:
      - postgres_primary_data:/var/lib/postgresql/data
      - ./postgres/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./postgres/pg_hba.conf:/etc/postgresql/pg_hba.conf
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    networks:
      - backend-network
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G

  postgres-replica:
    image: postgres:14-alpine
    environment:
      POSTGRES_REPLICATION_MODE: slave
      POSTGRES_MASTER_HOST: postgres-primary
      POSTGRES_REPLICATION_USER: replicator
      POSTGRES_REPLICATION_PASSWORD: ${REPLICATION_PASSWORD}
    volumes:
      - postgres_replica_data:/var/lib/postgresql/data
    depends_on:
      - postgres-primary
    networks:
      - backend-network
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 2G

  redis-cluster:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    networks:
      - backend-network
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 1G

  api:
    image: envoy-control-plane:latest
    environment:
      DATABASE_URL: ${DATABASE_URL}
      DATABASE_READ_URL: ${DATABASE_READ_URL}
      REDIS_URL: ${REDIS_URL}
      NATS_URL: ${NATS_URL}
      JWT_SECRET: ${JWT_SECRET}
      RUST_LOG: ${RUST_LOG:-info}
      SERVER_PORT: 8080
    networks:
      - backend-network
      - frontend-network
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '1'
          memory: 2G

  xds-server:
    image: envoy-xds-server:latest
    environment:
      DATABASE_READ_URL: ${DATABASE_READ_URL}
      REDIS_URL: ${REDIS_URL}
      GRPC_PORT: 18000
    networks:
      - backend-network
      - data-plane-network
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    ports:
      - "443:443"
      - "80:80"
    networks:
      - frontend-network
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - monitoring-network
      - backend-network

  grafana:
    image: grafana/grafana:latest
    volumes:
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - monitoring-network

volumes:
  postgres_primary_data:
  postgres_replica_data:
  redis_data:
  prometheus_data:
  grafana_data:

networks:
  backend-network:
    driver: overlay
    encrypted: true
  frontend-network:
    driver: overlay
  data-plane-network:
    driver: overlay
    encrypted: true
  monitoring-network:
    driver: overlay
```

## 4. Kubernetes Deployment

### 4.1 Kubernetes Manifests

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: envoy-control
---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-control-config
  namespace: envoy-control
data:
  RUST_LOG: "info"
  SERVER_PORT: "8080"
  GRPC_PORT: "18000"
---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: envoy-control-secrets
  namespace: envoy-control
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:pass@postgres:5432/envoy_control"
  REDIS_URL: "redis://redis:6379"
  JWT_SECRET: "your-secret-key"
---
# k8s/deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: envoy-control
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
      - name: api
        image: envoy-control-plane:latest
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        envFrom:
        - configMapRef:
            name: envoy-control-config
        - secretRef:
            name: envoy-control-secrets
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
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
---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: envoy-control
spec:
  selector:
    app: api-server
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  type: ClusterIP
---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
  namespace: envoy-control
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
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
---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: envoy-control
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.envoy-control.example.com
    secretName: api-tls
  rules:
  - host: api.envoy-control.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### 4.2 Helm Chart

```yaml
# helm/envoy-control/values.yaml
replicaCount: 3

image:
  repository: envoy-control-plane
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: api.envoy-control.example.com
      paths:
        - path: /api
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.envoy-control.example.com

resources:
  limits:
    cpu: 1000m
    memory: 2Gi
  requests:
    cpu: 250m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

postgresql:
  enabled: true
  auth:
    postgresPassword: "changeme"
    database: "envoy_control"
  primary:
    persistence:
      size: 100Gi
  readReplicas:
    replicaCount: 2

redis:
  enabled: true
  auth:
    enabled: true
    password: "changeme"
  master:
    persistence:
      size: 10Gi
  replica:
    replicaCount: 2

monitoring:
  prometheus:
    enabled: true
  grafana:
    enabled: true
```

## 5. Scaling Strategies

### 5.1 Horizontal Scaling

```mermaid
graph TB
    subgraph "Load Balancer Layer"
        LB1[Load Balancer 1]
        LB2[Load Balancer 2]
    end

    subgraph "API Server Pool"
        API1[API Server 1]
        API2[API Server 2]
        API3[API Server 3]
        APIN[API Server N]
    end

    subgraph "Database Layer"
        PG_PRIMARY[(PostgreSQL Primary)]
        PG_REPLICA1[(PostgreSQL Replica 1)]
        PG_REPLICA2[(PostgreSQL Replica 2)]
    end

    subgraph "Cache Layer"
        REDIS1[Redis Node 1]
        REDIS2[Redis Node 2]
        REDIS3[Redis Node 3]
    end

    LB1 --> API1
    LB1 --> API2
    LB2 --> API3
    LB2 --> APIN

    API1 --> PG_PRIMARY
    API2 --> PG_REPLICA1
    API3 --> PG_REPLICA2
    APIN --> PG_REPLICA1

    API1 --> REDIS1
    API2 --> REDIS2
    API3 --> REDIS3
```

### 5.2 Auto-Scaling Configuration

```yaml
# Auto-scaling policies
apiVersion: v1
kind: ConfigMap
metadata:
  name: scaling-policies
data:
  scaling_rules.yaml: |
    rules:
      - name: api_server_scale_up
        conditions:
          - metric: cpu_utilization
            operator: ">"
            threshold: 70
            duration: 2m
          - metric: memory_utilization
            operator: ">"
            threshold: 80
            duration: 2m
          - metric: request_rate
            operator: ">"
            threshold: 1000
            duration: 1m
        action:
          type: scale_up
          increment: 2
          max_instances: 20
          cooldown: 5m

      - name: api_server_scale_down
        conditions:
          - metric: cpu_utilization
            operator: "<"
            threshold: 30
            duration: 10m
          - metric: memory_utilization
            operator: "<"
            threshold: 40
            duration: 10m
        action:
          type: scale_down
          decrement: 1
          min_instances: 3
          cooldown: 10m

      - name: database_replica_scale
        conditions:
          - metric: read_query_queue
            operator: ">"
            threshold: 100
            duration: 5m
        action:
          type: add_replica
          max_replicas: 5
```

### 5.3 Database Scaling

```sql
-- Partitioning strategy for large tables
CREATE TABLE proxy_metrics_2024_01 PARTITION OF proxy_metrics
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE proxy_metrics_2024_02 PARTITION OF proxy_metrics
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Index optimization for performance
CREATE INDEX CONCURRENTLY idx_proxies_tenant_status
    ON envoy_proxies(tenant_id, status)
    WHERE status IN ('active', 'healthy');

-- Connection pooling configuration
ALTER SYSTEM SET max_connections = 200;
ALTER SYSTEM SET shared_buffers = '4GB';
ALTER SYSTEM SET effective_cache_size = '12GB';
ALTER SYSTEM SET work_mem = '16MB';
```

## 6. Load Balancing

### 6.1 NGINX Configuration

```nginx
# nginx/nginx.conf
upstream api_servers {
    least_conn;

    server api-1:8080 max_fails=3 fail_timeout=30s;
    server api-2:8080 max_fails=3 fail_timeout=30s;
    server api-3:8080 max_fails=3 fail_timeout=30s;

    keepalive 32;
}

upstream xds_servers {
    least_conn;

    server xds-1:18000;
    server xds-2:18000;

    keepalive 16;
}

server {
    listen 443 ssl http2;
    server_name api.envoy-control.example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.3;

    # API endpoints
    location /api/ {
        proxy_pass http://api_servers;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Buffering
        proxy_buffering off;

        # Rate limiting
        limit_req zone=api_limit burst=100 nodelay;
    }

    # WebSocket support
    location /api/v1/ws {
        proxy_pass http://api_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
    }

    # Health check endpoint
    location /health {
        access_log off;
        proxy_pass http://api_servers/health;
    }
}

# gRPC for xDS
server {
    listen 18000 http2;

    location / {
        grpc_pass grpc://xds_servers;

        # gRPC specific settings
        grpc_read_timeout 300;
        grpc_send_timeout 300;
        client_body_timeout 60;
        client_body_buffer_size 8k;
        grpc_buffer_size 64k;
    }
}
```

## 7. Monitoring & Observability

### 7.1 Prometheus Configuration

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'api-servers'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - envoy-control
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - '/etc/prometheus/alerts/*.yml'
```

### 7.2 Alert Rules

```yaml
# prometheus/alerts/api.yml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is above 5% for {{ $labels.instance }}"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency"
          description: "P99 latency is above 500ms"

      - alert: PodMemoryUsage
        expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod memory usage high"
          description: "Pod {{ $labels.pod }} memory usage is above 90%"
```

### 7.3 Grafana Dashboards

```json
{
  "dashboard": {
    "title": "Envoy Control Plane",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m])"
          }
        ]
      },
      {
        "title": "P99 Latency",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Active Connections",
        "targets": [
          {
            "expr": "http_connections_active"
          }
        ]
      }
    ]
  }
}
```

## 8. Disaster Recovery

### 8.1 Backup Strategy

```bash
#!/bin/bash
# backup.sh

# Database backup
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/postgres"
S3_BUCKET="s3://envoy-control-backups"

# Full backup
pg_dump $DATABASE_URL | gzip > $BACKUP_DIR/full_backup_$DATE.sql.gz

# Upload to S3
aws s3 cp $BACKUP_DIR/full_backup_$DATE.sql.gz $S3_BUCKET/postgres/

# WAL archiving
archive_command = 'test ! -f /backup/wal/%f && cp %p /backup/wal/%f'

# Redis backup
redis-cli --rdb $BACKUP_DIR/redis_backup_$DATE.rdb
aws s3 cp $BACKUP_DIR/redis_backup_$DATE.rdb $S3_BUCKET/redis/

# Configuration backup
kubectl get configmaps -n envoy-control -o yaml > $BACKUP_DIR/configmaps_$DATE.yaml
kubectl get secrets -n envoy-control -o yaml > $BACKUP_DIR/secrets_$DATE.yaml
```

### 8.2 Recovery Procedures

```bash
#!/bin/bash
# restore.sh

# Restore database from backup
DATE=$1
BACKUP_FILE="full_backup_$DATE.sql.gz"

# Download from S3
aws s3 cp s3://envoy-control-backups/postgres/$BACKUP_FILE /tmp/

# Restore database
gunzip -c /tmp/$BACKUP_FILE | psql $DATABASE_URL

# Restore Redis
aws s3 cp s3://envoy-control-backups/redis/redis_backup_$DATE.rdb /tmp/
redis-cli --rdb /tmp/redis_backup_$DATE.rdb

# Apply configuration
kubectl apply -f configmaps_$DATE.yaml
kubectl apply -f secrets_$DATE.yaml
```

## 9. Zero-Downtime Deployment

### 9.1 Rolling Update Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  template:
    spec:
      containers:
      - name: api
        image: envoy-control-plane:v2.0.0
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
          failureThreshold: 3
```

### 9.2 Blue-Green Deployment

```bash
#!/bin/bash
# blue-green-deploy.sh

NEW_VERSION=$1
OLD_VERSION=$(kubectl get deployment api-server-blue -o jsonpath='{.spec.template.spec.containers[0].image}' | cut -d: -f2)

echo "Deploying Green environment with version $NEW_VERSION"

# Deploy to green
kubectl set image deployment/api-server-green api=envoy-control-plane:$NEW_VERSION

# Wait for green to be ready
kubectl wait --for=condition=available --timeout=300s deployment/api-server-green

# Run smoke tests
./run-smoke-tests.sh green

if [ $? -eq 0 ]; then
    echo "Switching traffic to Green"
    kubectl patch service api-service -p '{"spec":{"selector":{"version":"green"}}}'

    echo "Monitoring for 5 minutes"
    sleep 300

    # Check metrics
    ERROR_RATE=$(curl -s http://prometheus:9090/api/v1/query?query=rate(http_requests_total{status=~"5.."}[5m]) | jq '.data.result[0].value[1]')

    if (( $(echo "$ERROR_RATE < 0.01" | bc -l) )); then
        echo "Deployment successful, updating Blue"
        kubectl set image deployment/api-server-blue api=envoy-control-plane:$NEW_VERSION
    else
        echo "High error rate detected, rolling back"
        kubectl patch service api-service -p '{"spec":{"selector":{"version":"blue"}}}'
    fi
else
    echo "Smoke tests failed, aborting deployment"
    exit 1
fi
```

## 10. Performance Optimization

### 10.1 Application Tuning

```rust
// src/main.rs - Axum server configuration
use axum::extract::DefaultBodyLimit;
use tower::ServiceBuilder;
use tower_http::compression::CompressionLayer;

#[tokio::main]
async fn main() {
    // Configure runtime
    let runtime = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(num_cpus::get())
        .thread_name("axum-worker")
        .enable_all()
        .build()
        .unwrap();

    // SQLx connection pool
    let pool = PgPoolOptions::new()
        .max_connections(100)
        .min_connections(10)
        .connect_timeout(Duration::from_secs(30))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .connect(&database_url)
        .await
        .unwrap();

    // Redis connection pool
    let redis_pool = redis::Client::open(redis_url)
        .unwrap()
        .get_multiplexed_tokio_connection()
        .await
        .unwrap();

    // Build application
    let app = Router::new()
        .route("/api/v1/*path", api_routes())
        .layer(
            ServiceBuilder::new()
                .layer(CompressionLayer::new())
                .layer(DefaultBodyLimit::max(10 * 1024 * 1024)) // 10MB
                .layer(tower_http::trace::TraceLayer::new_for_http())
                .layer(tower_http::timeout::TimeoutLayer::new(Duration::from_secs(30)))
        );

    // Start server
    let addr = SocketAddr::from(([0, 0, 0, 0], 8080));
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

### 10.2 Database Optimization

```sql
-- Optimize PostgreSQL configuration
ALTER SYSTEM SET shared_buffers = '25% of RAM';
ALTER SYSTEM SET effective_cache_size = '75% of RAM';
ALTER SYSTEM SET maintenance_work_mem = '256MB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET default_statistics_target = 100;
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;
ALTER SYSTEM SET work_mem = '16MB';
ALTER SYSTEM SET min_wal_size = '1GB';
ALTER SYSTEM SET max_wal_size = '4GB';
```

## 11. Security Hardening

### 11.1 Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-network-policy
  namespace: envoy-control
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: envoy-control
    - podSelector:
        matchLabels:
          app: nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: envoy-control
    ports:
    - protocol: TCP
      port: 5432  # PostgreSQL
    - protocol: TCP
      port: 6379  # Redis
    - protocol: TCP
      port: 4222  # NATS
```

### 11.2 Pod Security Policy

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  readOnlyRootFilesystem: true
```

## 12. CI/CD Pipeline

### 12.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable

      - name: Run tests
        run: |
          cargo test
          cargo test --integration

      - name: Run security audit
        run: cargo audit

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build Docker image
        run: |
          docker build -t envoy-control-plane:${{ github.sha }} .

      - name: Push to registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
          docker tag envoy-control-plane:${{ github.sha }} ${{ secrets.DOCKER_REGISTRY }}/envoy-control-plane:${{ github.sha }}
          docker push ${{ secrets.DOCKER_REGISTRY }}/envoy-control-plane:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/api-server api=${{ secrets.DOCKER_REGISTRY }}/envoy-control-plane:${{ github.sha }}
          kubectl rollout status deployment/api-server
```

## 13. Conclusion

This deployment and scaling strategy provides:

1. **High Availability**: Multi-replica deployments with health checks
2. **Scalability**: Auto-scaling based on metrics
3. **Performance**: Optimized configurations and caching
4. **Security**: Network policies and security hardening
5. **Observability**: Comprehensive monitoring and alerting
6. **Reliability**: Backup strategies and disaster recovery
7. **Zero-downtime**: Rolling updates and blue-green deployments

The system is designed to handle 10,000+ Envoy proxy instances with sub-100ms response times and 99.9% availability.