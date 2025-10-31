# Envoy Proxy Fleet Implementation Plan
## Docker-based Standalone Fleet with Per-Instance Configuration

**Document Version**: 1.0
**Last Updated**: 2025-10-30
**Status**: Implementation Plan
**Authors**: System Architecture Team

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Fleet Architecture](#2-fleet-architecture)
3. [Docker Container Design](#3-docker-container-design)
4. [Per-Instance Configuration System](#4-per-instance-configuration-system)
5. [Fleet Management Components](#5-fleet-management-components)
6. [Implementation Phases](#6-implementation-phases)
7. [Configuration Examples](#7-configuration-examples)
8. [Deployment and Operations](#8-deployment-and-operations)
9. [Testing Strategy](#9-testing-strategy)
10. [Monitoring and Observability](#10-monitoring-and-observability)

---

## 1. Executive Summary

This document provides a detailed implementation plan for deploying a standalone fleet of Envoy proxy instances in Docker containers, where each instance has its own unique configuration managed by the Rust-based control plane.

### 1.1 Key Design Principles

- **Instance Isolation**: Each Envoy proxy runs in its own Docker container with dedicated resources
- **Dynamic Configuration**: Per-instance configuration delivered via xDS protocol
- **Zero Downtime**: Rolling updates with health checks and graceful shutdown
- **Scalability**: Support for 100+ concurrent Envoy instances on a single host
- **Observability**: Per-instance metrics, logging, and tracing
- **Security**: mTLS communication, isolated networks, minimal attack surface

### 1.2 Architecture Overview

```mermaid
graph TB
    subgraph "Control Plane"
        CP[Control Plane<br/>Rust xDS Server]
        CONFIG[Config Manager]
        DB[(PostgreSQL<br/>Config Store)]
    end

    subgraph "Docker Host"
        subgraph "Network: envoy-fleet"
            E1[Envoy Container 1<br/>node-id: proxy-001<br/>Port: 10001]
            E2[Envoy Container 2<br/>node-id: proxy-002<br/>Port: 10002]
            E3[Envoy Container 3<br/>node-id: proxy-003<br/>Port: 10003]
            EN[Envoy Container N<br/>node-id: proxy-00N<br/>Port: 1000N]
        end
    end

    subgraph "Backend Services"
        S1[Service A]
        S2[Service B]
        S3[Service C]
    end

    CP <-->|xDS gRPC| E1
    CP <-->|xDS gRPC| E2
    CP <-->|xDS gRPC| E3
    CP <-->|xDS gRPC| EN

    CONFIG --> DB
    CONFIG --> CP

    E1 --> S1
    E1 --> S2
    E2 --> S2
    E2 --> S3
    E3 --> S1
    E3 --> S3
    EN --> S1
```

### 1.3 Goals and Objectives

- **Primary Goal**: Create a fully functional Docker-based Envoy fleet with independent configuration per instance
- **Scale Target**: Support 100+ Envoy instances per host
- **Performance Target**: Sub-second configuration propagation
- **Reliability Target**: 99.9% uptime for individual proxies
- **Operational Target**: Simple deployment and management via Docker Compose or CLI tools

---

## 2. Fleet Architecture

### 2.1 System Components

```mermaid
graph LR
    subgraph "Management Layer"
        CLI[Fleet CLI Tool]
        API[Management API]
    end

    subgraph "Control Plane"
        XDS[xDS Server]
        CM[Config Manager]
        FM[Fleet Manager]
        HC[Health Monitor]
    end

    subgraph "Container Orchestration"
        DE[Docker Engine]
        DN[Docker Network]
        DV[Docker Volumes]
    end

    subgraph "Envoy Fleet"
        E1[Envoy 1]
        E2[Envoy 2]
        EN[Envoy N]
    end

    CLI --> API
    API --> FM
    FM --> CM
    CM --> XDS
    XDS --> E1
    XDS --> E2
    XDS --> EN
    FM --> DE
    DE --> E1
    DE --> E2
    DE --> EN
    HC --> E1
    HC --> E2
    HC --> EN
```

### 2.2 Component Responsibilities

#### 2.2.1 Fleet Manager (Rust)
- Creates and manages Envoy Docker containers
- Assigns unique node IDs and ports to each instance
- Manages container lifecycle (start, stop, restart, remove)
- Monitors container health and status
- Handles container scaling (add/remove instances)

#### 2.2.2 Configuration Manager (Rust)
- Stores per-instance configuration in PostgreSQL
- Validates configuration before deployment
- Generates xDS snapshots for each instance
- Manages configuration versioning and rollback
- Provides configuration templates and inheritance

#### 2.2.3 xDS Server (Rust)
- Implements Envoy xDS v3 protocol
- Maintains persistent gRPC streams per instance
- Delivers instance-specific configuration
- Handles ACK/NACK responses
- Tracks configuration application status

#### 2.2.4 Health Monitor (Rust)
- Monitors Envoy container health
- Tracks xDS connection status
- Collects metrics from each instance
- Detects and alerts on failures
- Triggers automatic recovery actions

---

## 3. Docker Container Design

### 3.1 Envoy Container Specification

#### 3.1.1 Base Dockerfile

```dockerfile
# Dockerfile.envoy
FROM envoyproxy/envoy:v1.31-latest

# Install required tools
RUN apt-get update && \
    apt-get install -y curl jq && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Create necessary directories
RUN mkdir -p /etc/envoy /var/log/envoy /var/run/envoy

# Add bootstrap configuration template
COPY bootstrap-template.yaml /etc/envoy/bootstrap-template.yaml

# Add entrypoint script
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

# Expose admin port (19000) and proxy port (10000)
EXPOSE 19000 10000

# Health check
HEALTHCHECK --interval=5s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:19000/ready || exit 1

# Set entrypoint
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

#### 3.1.2 Bootstrap Configuration Template

```yaml
# bootstrap-template.yaml
node:
  id: ${NODE_ID}
  cluster: ${CLUSTER_NAME}
  locality:
    zone: ${ZONE}
  metadata:
    instance_id: ${INSTANCE_ID}
    version: ${VERSION}
    environment: ${ENVIRONMENT}

admin:
  access_log_path: /var/log/envoy/admin.log
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 19000

dynamic_resources:
  ads_config:
    api_type: GRPC
    transport_api_version: V3
    grpc_services:
      - envoy_grpc:
          cluster_name: xds_cluster
    set_node_on_first_message_only: true

  lds_config:
    resource_api_version: V3
    ads: {}

  cds_config:
    resource_api_version: V3
    ads: {}

static_resources:
  clusters:
    - name: xds_cluster
      type: STRICT_DNS
      typed_extension_protocol_options:
        envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
          "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
          explicit_http_config:
            http2_protocol_options: {}
      load_assignment:
        cluster_name: xds_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: ${CONTROL_PLANE_HOST}
                      port_value: ${CONTROL_PLANE_PORT}
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
          common_tls_context:
            tls_certificates:
              - certificate_chain:
                  filename: /etc/envoy/certs/client-cert.pem
                - private_key:
                  filename: /etc/envoy/certs/client-key.pem
            validation_context:
              trusted_ca:
                filename: /etc/envoy/certs/ca-cert.pem

layered_runtime:
  layers:
    - name: static_layer
      static_layer:
        overload:
          global_downstream_max_connections: 50000
```

#### 3.1.3 Entrypoint Script

```bash
#!/bin/bash
# entrypoint.sh

set -e

# Required environment variables
: ${NODE_ID:?NODE_ID is required}
: ${CONTROL_PLANE_HOST:?CONTROL_PLANE_HOST is required}
: ${CONTROL_PLANE_PORT:-18000}
: ${CLUSTER_NAME:-default-cluster}
: ${ZONE:-local}
: ${INSTANCE_ID:-${HOSTNAME}}
: ${VERSION:-v1}
: ${ENVIRONMENT:-development}

# Generate bootstrap config from template
echo "Generating bootstrap configuration for node ${NODE_ID}"
envsubst < /etc/envoy/bootstrap-template.yaml > /etc/envoy/bootstrap.yaml

# Verify bootstrap config
echo "Verifying bootstrap configuration..."
if ! /usr/local/bin/envoy --mode validate -c /etc/envoy/bootstrap.yaml; then
    echo "Bootstrap configuration validation failed!"
    exit 1
fi

echo "Starting Envoy proxy with node ID: ${NODE_ID}"
exec /usr/local/bin/envoy -c /etc/envoy/bootstrap.yaml \
    --service-cluster ${CLUSTER_NAME} \
    --service-node ${NODE_ID} \
    --log-level ${LOG_LEVEL:-info} \
    --log-format '[%Y-%m-%d %T.%e][%t][%l][%n] %v'
```

### 3.2 Container Naming and Networking

#### 3.2.1 Naming Convention

```
envoy-proxy-<cluster>-<instance-id>

Examples:
- envoy-proxy-edge-001
- envoy-proxy-edge-002
- envoy-proxy-internal-001
- envoy-proxy-internal-002
```

#### 3.2.2 Network Configuration

```yaml
# Docker network configuration
networks:
  envoy-fleet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
    driver_opts:
      com.docker.network.bridge.name: envoy-fleet-br
      com.docker.network.driver.mtu: 1500

  backend-services:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.0.0/16
          gateway: 172.21.0.1
```

#### 3.2.3 Port Allocation Strategy

```
Base Port Range: 10000-19999 (proxy ports)
Admin Port Range: 20000-29999 (admin interfaces)

Instance N:
- Proxy Port: 10000 + N
- Admin Port: 20000 + N

Example for instance 5:
- Proxy Port: 10005
- Admin Port: 20005
```

### 3.3 Resource Limits

```yaml
# Per-container resource limits
resources:
  limits:
    cpus: '0.5'          # 0.5 CPU core
    memory: 256M          # 256 MB RAM
  reservations:
    cpus: '0.25'         # Minimum 0.25 CPU
    memory: 128M          # Minimum 128 MB RAM

# For high-traffic instances
resources_high:
  limits:
    cpus: '2.0'          # 2 CPU cores
    memory: 1G            # 1 GB RAM
  reservations:
    cpus: '1.0'          # Minimum 1 CPU
    memory: 512M          # Minimum 512 MB RAM
```

---

## 4. Per-Instance Configuration System

### 4.1 Configuration Storage Schema

#### 4.1.1 PostgreSQL Database Schema

```sql
-- Fleet instances table
CREATE TABLE envoy_instances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id VARCHAR(255) UNIQUE NOT NULL,
    cluster_name VARCHAR(255) NOT NULL,
    container_id VARCHAR(255) UNIQUE,
    container_name VARCHAR(255) UNIQUE NOT NULL,
    proxy_port INTEGER NOT NULL,
    admin_port INTEGER NOT NULL,
    status VARCHAR(50) NOT NULL, -- running, stopped, failed, starting
    health_status VARCHAR(50), -- healthy, unhealthy, unknown
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    last_seen_at TIMESTAMP,
    metadata JSONB,
    CONSTRAINT unique_proxy_port UNIQUE (proxy_port),
    CONSTRAINT unique_admin_port UNIQUE (admin_port)
);

-- Instance configurations table
CREATE TABLE instance_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instance_id UUID NOT NULL REFERENCES envoy_instances(id) ON DELETE CASCADE,
    config_version VARCHAR(100) NOT NULL,
    config_type VARCHAR(50) NOT NULL, -- listener, route, cluster, endpoint, secret
    config_name VARCHAR(255) NOT NULL,
    config_data JSONB NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by VARCHAR(255),
    CONSTRAINT unique_instance_config UNIQUE (instance_id, config_type, config_name)
);

-- Configuration history table
CREATE TABLE config_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instance_id UUID NOT NULL REFERENCES envoy_instances(id) ON DELETE CASCADE,
    config_version VARCHAR(100) NOT NULL,
    snapshot_data JSONB NOT NULL,
    applied_at TIMESTAMP NOT NULL DEFAULT NOW(),
    applied_by VARCHAR(255),
    status VARCHAR(50) NOT NULL, -- pending, applied, rejected, rolled_back
    ack_response JSONB,
    nack_reason TEXT
);

-- Configuration templates table
CREATE TABLE config_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_name VARCHAR(255) UNIQUE NOT NULL,
    template_type VARCHAR(50) NOT NULL,
    template_data JSONB NOT NULL,
    description TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_instances_node_id ON envoy_instances(node_id);
CREATE INDEX idx_instances_status ON envoy_instances(status);
CREATE INDEX idx_instances_cluster ON envoy_instances(cluster_name);
CREATE INDEX idx_configs_instance ON instance_configs(instance_id);
CREATE INDEX idx_configs_version ON instance_configs(config_version);
CREATE INDEX idx_history_instance ON config_history(instance_id);
CREATE INDEX idx_history_applied_at ON config_history(applied_at);
```

### 4.2 Configuration Data Structures

#### 4.2.1 Rust Configuration Model

```rust
// src/models/instance.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};
use std::collections::HashMap;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EnvoyInstance {
    pub id: Uuid,
    pub node_id: String,
    pub cluster_name: String,
    pub container_id: Option<String>,
    pub container_name: String,
    pub proxy_port: u16,
    pub admin_port: u16,
    pub status: InstanceStatus,
    pub health_status: Option<HealthStatus>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub last_seen_at: Option<DateTime<Utc>>,
    pub metadata: HashMap<String, serde_json::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum InstanceStatus {
    Running,
    Stopped,
    Failed,
    Starting,
    Stopping,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum HealthStatus {
    Healthy,
    Unhealthy,
    Unknown,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct InstanceConfig {
    pub id: Uuid,
    pub instance_id: Uuid,
    pub config_version: String,
    pub config_type: ConfigType,
    pub config_name: String,
    pub config_data: serde_json::Value,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
    pub created_by: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum ConfigType {
    Listener,
    Route,
    Cluster,
    Endpoint,
    Secret,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConfigSnapshot {
    pub instance_id: Uuid,
    pub version: String,
    pub listeners: Vec<ListenerConfig>,
    pub routes: Vec<RouteConfig>,
    pub clusters: Vec<ClusterConfig>,
    pub endpoints: Vec<EndpointConfig>,
    pub secrets: Vec<SecretConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ListenerConfig {
    pub name: String,
    pub address: String,
    pub port: u16,
    pub filter_chains: Vec<FilterChain>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RouteConfig {
    pub name: String,
    pub virtual_hosts: Vec<VirtualHost>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ClusterConfig {
    pub name: String,
    pub cluster_type: ClusterType,
    pub load_assignment: Option<LoadAssignment>,
    pub health_checks: Vec<HealthCheck>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EndpointConfig {
    pub cluster_name: String,
    pub endpoints: Vec<Endpoint>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SecretConfig {
    pub name: String,
    pub tls_certificate: TlsCertificate,
}
```

### 4.3 Configuration Templates

#### 4.3.1 Template System Design

```rust
// src/config/templates.rs
use handlebars::Handlebars;
use serde_json::json;

pub struct ConfigTemplateEngine {
    handlebars: Handlebars<'static>,
}

impl ConfigTemplateEngine {
    pub fn new() -> Self {
        let mut handlebars = Handlebars::new();

        // Register built-in templates
        handlebars
            .register_template_string("http_listener", include_str!("templates/http_listener.hbs"))
            .unwrap();
        handlebars
            .register_template_string("tcp_listener", include_str!("templates/tcp_listener.hbs"))
            .unwrap();
        handlebars
            .register_template_string("http_route", include_str!("templates/http_route.hbs"))
            .unwrap();
        handlebars
            .register_template_string("cluster", include_str!("templates/cluster.hbs"))
            .unwrap();

        Self { handlebars }
    }

    pub fn render_listener(
        &self,
        template_name: &str,
        params: &ListenerTemplateParams,
    ) -> Result<serde_json::Value, Box<dyn std::error::Error>> {
        let rendered = self.handlebars.render(template_name, params)?;
        Ok(serde_json::from_str(&rendered)?)
    }
}

#[derive(Debug, Serialize)]
pub struct ListenerTemplateParams {
    pub name: String,
    pub address: String,
    pub port: u16,
    pub route_config_name: String,
    pub use_tls: bool,
    pub tls_secret_name: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct RouteTemplateParams {
    pub name: String,
    pub domain: String,
    pub routes: Vec<RouteMatch>,
}

#[derive(Debug, Serialize)]
pub struct RouteMatch {
    pub path_prefix: String,
    pub cluster_name: String,
    pub timeout_seconds: u32,
}

#[derive(Debug, Serialize)]
pub struct ClusterTemplateParams {
    pub name: String,
    pub lb_policy: String, // ROUND_ROBIN, LEAST_REQUEST, RING_HASH
    pub use_eds: bool,
    pub health_check_path: Option<String>,
    pub health_check_interval_seconds: Option<u32>,
}
```

#### 4.3.2 Example Templates

```handlebars
{{!-- templates/http_listener.hbs --}}
{
  "name": "{{name}}",
  "address": {
    "socket_address": {
      "address": "{{address}}",
      "port_value": {{port}}
    }
  },
  "filter_chains": [
    {
      "filters": [
        {
          "name": "envoy.filters.network.http_connection_manager",
          "typed_config": {
            "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
            "stat_prefix": "ingress_http",
            "codec_type": "AUTO",
            "route_config": {
              "name": "{{route_config_name}}"
            },
            "http_filters": [
              {
                "name": "envoy.filters.http.router",
                "typed_config": {
                  "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                }
              }
            ]
          }
        }
      ]
      {{#if use_tls}}
      ,
      "transport_socket": {
        "name": "envoy.transport_sockets.tls",
        "typed_config": {
          "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext",
          "common_tls_context": {
            "tls_certificate_sds_secret_configs": [
              {
                "name": "{{tls_secret_name}}",
                "sds_config": {
                  "resource_api_version": "V3",
                  "ads": {}
                }
              }
            ]
          }
        }
      }
      {{/if}}
    }
  ]
}
```

### 4.4 Configuration Assignment Strategy

#### 4.4.1 Assignment Modes

```rust
// src/config/assignment.rs

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ConfigAssignmentMode {
    /// Each instance gets a unique configuration
    PerInstance,

    /// Instances in the same group share configuration
    ByGroup { group_name: String },

    /// Configuration based on instance metadata
    ByMetadata {
        match_key: String,
        match_value: String,
    },

    /// Template-based with parameter substitution
    Template {
        template_name: String,
        parameters: HashMap<String, serde_json::Value>,
    },

    /// Inherit from parent with overrides
    Inherit {
        parent_instance_id: Uuid,
        overrides: Vec<ConfigOverride>,
    },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConfigOverride {
    pub config_type: ConfigType,
    pub config_name: String,
    pub override_data: serde_json::Value,
}

pub struct ConfigAssignmentEngine {
    db_pool: sqlx::PgPool,
    template_engine: ConfigTemplateEngine,
}

impl ConfigAssignmentEngine {
    pub async fn assign_config(
        &self,
        instance_id: Uuid,
        mode: ConfigAssignmentMode,
    ) -> Result<ConfigSnapshot, Box<dyn std::error::Error>> {
        match mode {
            ConfigAssignmentMode::PerInstance => {
                self.assign_per_instance_config(instance_id).await
            }
            ConfigAssignmentMode::ByGroup { group_name } => {
                self.assign_group_config(instance_id, &group_name).await
            }
            ConfigAssignmentMode::ByMetadata { match_key, match_value } => {
                self.assign_metadata_config(instance_id, &match_key, &match_value).await
            }
            ConfigAssignmentMode::Template { template_name, parameters } => {
                self.assign_template_config(instance_id, &template_name, parameters).await
            }
            ConfigAssignmentMode::Inherit { parent_instance_id, overrides } => {
                self.assign_inherited_config(instance_id, parent_instance_id, overrides).await
            }
        }
    }
}
```

---

## 5. Fleet Management Components

### 5.1 Fleet Manager Implementation

#### 5.1.1 Core Fleet Manager

```rust
// src/fleet/manager.rs
use bollard::Docker;
use bollard::container::{Config, CreateContainerOptions, StartContainerOptions};
use bollard::models::{HostConfig, PortBinding};
use std::collections::HashMap;
use anyhow::Result;

pub struct FleetManager {
    docker: Docker,
    db_pool: sqlx::PgPool,
    config_manager: Arc<ConfigManager>,
    port_allocator: Arc<Mutex<PortAllocator>>,
}

impl FleetManager {
    pub async fn new(
        docker: Docker,
        db_pool: sqlx::PgPool,
        config_manager: Arc<ConfigManager>,
    ) -> Result<Self> {
        let port_allocator = Arc::new(Mutex::new(PortAllocator::new(10000, 30000)));

        Ok(Self {
            docker,
            db_pool,
            config_manager,
            port_allocator,
        })
    }

    /// Create a new Envoy proxy instance
    pub async fn create_instance(
        &self,
        params: CreateInstanceParams,
    ) -> Result<EnvoyInstance> {
        // Allocate ports
        let (proxy_port, admin_port) = self.allocate_ports().await?;

        // Generate unique node ID
        let node_id = format!("{}-{:03}", params.cluster_name, params.instance_number);
        let container_name = format!("envoy-proxy-{}", node_id);

        // Prepare environment variables
        let env_vars = vec![
            format!("NODE_ID={}", node_id),
            format!("CONTROL_PLANE_HOST={}", params.control_plane_host),
            format!("CONTROL_PLANE_PORT={}", params.control_plane_port),
            format!("CLUSTER_NAME={}", params.cluster_name),
            format!("ZONE={}", params.zone.unwrap_or_else(|| "local".to_string())),
            format!("ENVIRONMENT={}", params.environment.unwrap_or_else(|| "dev".to_string())),
        ];

        // Configure port bindings
        let mut port_bindings = HashMap::new();
        port_bindings.insert(
            "10000/tcp".to_string(),
            Some(vec![PortBinding {
                host_ip: Some("0.0.0.0".to_string()),
                host_port: Some(proxy_port.to_string()),
            }]),
        );
        port_bindings.insert(
            "19000/tcp".to_string(),
            Some(vec![PortBinding {
                host_ip: Some("127.0.0.1".to_string()),
                host_port: Some(admin_port.to_string()),
            }]),
        );

        // Create container configuration
        let host_config = HostConfig {
            port_bindings: Some(port_bindings),
            network_mode: Some("envoy-fleet".to_string()),
            restart_policy: Some(bollard::models::RestartPolicy {
                name: Some(bollard::models::RestartPolicyNameEnum::UNLESS_STOPPED),
                maximum_retry_count: None,
            }),
            memory: Some(268435456), // 256 MB
            nano_cpus: Some(500000000), // 0.5 CPU
            ..Default::default()
        };

        let config = Config {
            image: Some(params.image.unwrap_or_else(|| "envoyproxy/envoy:v1.31-latest".to_string())),
            env: Some(env_vars),
            hostname: Some(node_id.clone()),
            host_config: Some(host_config),
            labels: Some(params.labels.unwrap_or_default()),
            ..Default::default()
        };

        // Create container
        let create_options = CreateContainerOptions {
            name: container_name.clone(),
            platform: None,
        };

        let container = self.docker
            .create_container(Some(create_options), config)
            .await?;

        // Save instance to database
        let instance = sqlx::query_as::<_, EnvoyInstance>(
            r#"
            INSERT INTO envoy_instances
            (node_id, cluster_name, container_id, container_name, proxy_port, admin_port, status, metadata)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
            RETURNING *
            "#,
        )
        .bind(&node_id)
        .bind(&params.cluster_name)
        .bind(&container.id)
        .bind(&container_name)
        .bind(proxy_port as i32)
        .bind(admin_port as i32)
        .bind(InstanceStatus::Stopped)
        .bind(sqlx::types::Json(params.metadata.unwrap_or_default()))
        .fetch_one(&self.db_pool)
        .await?;

        tracing::info!(
            instance_id = %instance.id,
            node_id = %node_id,
            container_id = %container.id,
            "Created Envoy instance"
        );

        Ok(instance)
    }

    /// Start an Envoy proxy instance
    pub async fn start_instance(&self, instance_id: Uuid) -> Result<()> {
        let instance = self.get_instance(instance_id).await?;

        if let Some(container_id) = &instance.container_id {
            // Start container
            self.docker
                .start_container(container_id, None::<StartContainerOptions<String>>)
                .await?;

            // Update status
            sqlx::query(
                "UPDATE envoy_instances SET status = $1, updated_at = NOW() WHERE id = $2"
            )
            .bind(InstanceStatus::Running)
            .bind(instance_id)
            .execute(&self.db_pool)
            .await?;

            tracing::info!(
                instance_id = %instance_id,
                node_id = %instance.node_id,
                "Started Envoy instance"
            );

            Ok(())
        } else {
            Err(anyhow::anyhow!("Instance has no container ID"))
        }
    }

    /// Stop an Envoy proxy instance
    pub async fn stop_instance(&self, instance_id: Uuid) -> Result<()> {
        let instance = self.get_instance(instance_id).await?;

        if let Some(container_id) = &instance.container_id {
            // Graceful stop with 30 second timeout
            self.docker
                .stop_container(container_id, Some(bollard::container::StopContainerOptions {
                    t: 30,
                }))
                .await?;

            // Update status
            sqlx::query(
                "UPDATE envoy_instances SET status = $1, updated_at = NOW() WHERE id = $2"
            )
            .bind(InstanceStatus::Stopped)
            .bind(instance_id)
            .execute(&self.db_pool)
            .await?;

            tracing::info!(
                instance_id = %instance_id,
                node_id = %instance.node_id,
                "Stopped Envoy instance"
            );

            Ok(())
        } else {
            Err(anyhow::anyhow!("Instance has no container ID"))
        }
    }

    /// Remove an Envoy proxy instance
    pub async fn remove_instance(&self, instance_id: Uuid, force: bool) -> Result<()> {
        let instance = self.get_instance(instance_id).await?;

        if let Some(container_id) = &instance.container_id {
            // Stop if running
            if instance.status == InstanceStatus::Running {
                self.stop_instance(instance_id).await?;
            }

            // Remove container
            self.docker
                .remove_container(
                    container_id,
                    Some(bollard::container::RemoveContainerOptions {
                        force,
                        ..Default::default()
                    }),
                )
                .await?;

            // Delete from database (cascade will delete configs)
            sqlx::query("DELETE FROM envoy_instances WHERE id = $1")
                .bind(instance_id)
                .execute(&self.db_pool)
                .await?;

            // Release ports
            self.release_ports(instance.proxy_port, instance.admin_port).await?;

            tracing::info!(
                instance_id = %instance_id,
                node_id = %instance.node_id,
                "Removed Envoy instance"
            );

            Ok(())
        } else {
            Err(anyhow::anyhow!("Instance has no container ID"))
        }
    }

    /// List all instances
    pub async fn list_instances(
        &self,
        filter: Option<InstanceFilter>,
    ) -> Result<Vec<EnvoyInstance>> {
        let mut query = "SELECT * FROM envoy_instances WHERE 1=1".to_string();

        if let Some(f) = filter {
            if let Some(cluster) = f.cluster_name {
                query.push_str(&format!(" AND cluster_name = '{}'", cluster));
            }
            if let Some(status) = f.status {
                query.push_str(&format!(" AND status = '{}'", status));
            }
        }

        query.push_str(" ORDER BY created_at DESC");

        let instances = sqlx::query_as::<_, EnvoyInstance>(&query)
            .fetch_all(&self.db_pool)
            .await?;

        Ok(instances)
    }

    /// Scale fleet to target size
    pub async fn scale_fleet(
        &self,
        cluster_name: &str,
        target_count: usize,
    ) -> Result<Vec<EnvoyInstance>> {
        let current_instances = self.list_instances(Some(InstanceFilter {
            cluster_name: Some(cluster_name.to_string()),
            status: Some(InstanceStatus::Running),
        })).await?;

        let current_count = current_instances.len();

        if target_count > current_count {
            // Scale up
            let to_create = target_count - current_count;
            let mut new_instances = Vec::new();

            for i in 0..to_create {
                let instance_number = current_count + i + 1;
                let params = CreateInstanceParams {
                    cluster_name: cluster_name.to_string(),
                    instance_number,
                    control_plane_host: "host.docker.internal".to_string(),
                    control_plane_port: 18000,
                    ..Default::default()
                };

                let instance = self.create_instance(params).await?;
                self.start_instance(instance.id).await?;
                new_instances.push(instance);
            }

            tracing::info!(
                cluster = %cluster_name,
                added = to_create,
                "Scaled up fleet"
            );

            Ok(new_instances)
        } else if target_count < current_count {
            // Scale down
            let to_remove = current_count - target_count;
            let instances_to_remove: Vec<_> = current_instances
                .iter()
                .rev()
                .take(to_remove)
                .collect();

            for instance in instances_to_remove {
                self.remove_instance(instance.id, false).await?;
            }

            tracing::info!(
                cluster = %cluster_name,
                removed = to_remove,
                "Scaled down fleet"
            );

            Ok(vec![])
        } else {
            // No scaling needed
            Ok(vec![])
        }
    }

    // Helper methods
    async fn allocate_ports(&self) -> Result<(u16, u16)> {
        let mut allocator = self.port_allocator.lock().await;
        allocator.allocate_pair()
    }

    async fn release_ports(&self, proxy_port: u16, admin_port: u16) -> Result<()> {
        let mut allocator = self.port_allocator.lock().await;
        allocator.release(proxy_port);
        allocator.release(admin_port);
        Ok(())
    }

    async fn get_instance(&self, instance_id: Uuid) -> Result<EnvoyInstance> {
        sqlx::query_as::<_, EnvoyInstance>(
            "SELECT * FROM envoy_instances WHERE id = $1"
        )
        .bind(instance_id)
        .fetch_one(&self.db_pool)
        .await
        .map_err(|e| anyhow::anyhow!("Instance not found: {}", e))
    }
}

#[derive(Debug, Default)]
pub struct CreateInstanceParams {
    pub cluster_name: String,
    pub instance_number: usize,
    pub control_plane_host: String,
    pub control_plane_port: u16,
    pub image: Option<String>,
    pub zone: Option<String>,
    pub environment: Option<String>,
    pub labels: Option<HashMap<String, String>>,
    pub metadata: Option<HashMap<String, serde_json::Value>>,
}

#[derive(Debug, Default)]
pub struct InstanceFilter {
    pub cluster_name: Option<String>,
    pub status: Option<InstanceStatus>,
}

// Port allocator
struct PortAllocator {
    available_ports: std::collections::HashSet<u16>,
    min_port: u16,
    max_port: u16,
}

impl PortAllocator {
    fn new(min_port: u16, max_port: u16) -> Self {
        let available_ports = (min_port..=max_port).collect();
        Self {
            available_ports,
            min_port,
            max_port,
        }
    }

    fn allocate_pair(&mut self) -> Result<(u16, u16)> {
        let proxy_port = self.allocate_single()?;
        let admin_port = self.allocate_single()?;
        Ok((proxy_port, admin_port))
    }

    fn allocate_single(&mut self) -> Result<u16> {
        self.available_ports
            .iter()
            .next()
            .copied()
            .ok_or_else(|| anyhow::anyhow!("No available ports"))
            .map(|port| {
                self.available_ports.remove(&port);
                port
            })
    }

    fn release(&mut self, port: u16) {
        if port >= self.min_port && port <= self.max_port {
            self.available_ports.insert(port);
        }
    }
}
```

### 5.2 Health Monitoring System

```rust
// src/fleet/health.rs
use reqwest::Client;
use tokio::time::{interval, Duration};
use std::sync::Arc;

pub struct HealthMonitor {
    db_pool: sqlx::PgPool,
    http_client: Client,
    check_interval: Duration,
}

impl HealthMonitor {
    pub fn new(db_pool: sqlx::PgPool, check_interval_secs: u64) -> Self {
        Self {
            db_pool,
            http_client: Client::builder()
                .timeout(Duration::from_secs(5))
                .build()
                .unwrap(),
            check_interval: Duration::from_secs(check_interval_secs),
        }
    }

    pub async fn start(self: Arc<Self>) {
        let mut ticker = interval(self.check_interval);

        loop {
            ticker.tick().await;

            if let Err(e) = self.check_all_instances().await {
                tracing::error!(error = %e, "Health check failed");
            }
        }
    }

    async fn check_all_instances(&self) -> Result<()> {
        let instances = sqlx::query_as::<_, EnvoyInstance>(
            "SELECT * FROM envoy_instances WHERE status = 'running'"
        )
        .fetch_all(&self.db_pool)
        .await?;

        for instance in instances {
            tokio::spawn({
                let self_clone = self.clone();
                let instance_clone = instance.clone();
                async move {
                    if let Err(e) = self_clone.check_instance_health(&instance_clone).await {
                        tracing::warn!(
                            instance_id = %instance_clone.id,
                            node_id = %instance_clone.node_id,
                            error = %e,
                            "Instance health check failed"
                        );
                    }
                }
            });
        }

        Ok(())
    }

    async fn check_instance_health(&self, instance: &EnvoyInstance) -> Result<()> {
        let admin_url = format!("http://localhost:{}/ready", instance.admin_port);

        let health_status = match self.http_client.get(&admin_url).send().await {
            Ok(resp) if resp.status().is_success() => HealthStatus::Healthy,
            _ => HealthStatus::Unhealthy,
        };

        // Update health status and last seen
        sqlx::query(
            r#"
            UPDATE envoy_instances
            SET health_status = $1, last_seen_at = NOW(), updated_at = NOW()
            WHERE id = $2
            "#
        )
        .bind(health_status)
        .bind(instance.id)
        .execute(&self.db_pool)
        .await?;

        Ok(())
    }
}
```

---

## 6. Implementation Phases

### Phase 1: Docker Infrastructure (Week 1-2)

**Objectives:**
- Set up Docker environment and networking
- Create Envoy Docker image with bootstrap template
- Implement basic container lifecycle management

**Deliverables:**
1. Dockerfile for Envoy proxy
2. Bootstrap configuration template
3. Docker Compose file for testing
4. Basic fleet manager (create/start/stop/remove containers)

**Tasks:**
- [ ] Create Envoy Dockerfile with entrypoint script
- [ ] Set up Docker network configuration
- [ ] Implement port allocation system
- [ ] Create docker-compose.yml for local testing
- [ ] Write basic container management functions

### Phase 2: Configuration Storage (Week 3-4)

**Objectives:**
- Set up PostgreSQL database schema
- Implement configuration CRUD operations
- Create configuration template system

**Deliverables:**
1. PostgreSQL schema migrations
2. Configuration data models in Rust
3. Configuration storage layer
4. Template engine implementation

**Tasks:**
- [ ] Design and create database schema
- [ ] Implement Rust data models
- [ ] Create configuration storage trait and implementation
- [ ] Build template engine with Handlebars
- [ ] Write configuration validation logic

### Phase 3: Fleet Manager (Week 5-6)

**Objectives:**
- Complete fleet management functionality
- Implement instance lifecycle management
- Add fleet scaling capabilities

**Deliverables:**
1. Full FleetManager implementation
2. Instance creation and management
3. Scaling logic (scale up/down)
4. Health monitoring system

**Tasks:**
- [ ] Complete FleetManager struct with all methods
- [ ] Implement create/start/stop/remove instance operations
- [ ] Add fleet scaling functionality
- [ ] Build health monitoring service
- [ ] Create instance status tracking

### Phase 4: xDS Integration (Week 7-8)

**Objectives:**
- Integrate fleet with existing xDS server
- Implement per-instance configuration delivery
- Add configuration version management

**Deliverables:**
1. xDS server integration
2. Per-instance snapshot generation
3. Configuration versioning system
4. ACK/NACK handling

**Tasks:**
- [ ] Connect fleet manager to xDS server
- [ ] Implement snapshot generation per instance
- [ ] Add configuration version tracking
- [ ] Handle Envoy ACK/NACK responses
- [ ] Implement configuration rollback

### Phase 5: Management API (Week 9-10)

**Objectives:**
- Create REST API for fleet management
- Build CLI tool for fleet operations
- Add configuration assignment API

**Deliverables:**
1. REST API endpoints
2. CLI tool implementation
3. API documentation
4. Integration tests

**Tasks:**
- [ ] Implement REST API with Actix-web/Axum
- [ ] Create CLI tool with Clap
- [ ] Add API authentication and authorization
- [ ] Write comprehensive API documentation
- [ ] Create integration tests

### Phase 6: Testing and Optimization (Week 11-12)

**Objectives:**
- Comprehensive testing of fleet system
- Performance optimization
- Load testing with 100+ instances

**Deliverables:**
1. Test suite (unit + integration)
2. Load test scripts
3. Performance benchmarks
4. Documentation and examples

**Tasks:**
- [ ] Write unit tests for all components
- [ ] Create integration test scenarios
- [ ] Perform load testing with 100+ instances
- [ ] Optimize performance bottlenecks
- [ ] Write complete documentation

---

## 7. Configuration Examples

### 7.1 Docker Compose for Testing

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: envoy_control_plane
      POSTGRES_USER: controlplane
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - control-plane

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - control-plane

  control-plane:
    build: .
    depends_on:
      - postgres
      - redis
    environment:
      DATABASE_URL: postgresql://controlplane:secret@postgres/envoy_control_plane
      REDIS_URL: redis://redis:6379
      GRPC_PORT: 18000
      HTTP_PORT: 8080
    ports:
      - "18000:18000"  # gRPC xDS
      - "8080:8080"    # HTTP API
      - "9090:9090"    # Metrics
    networks:
      - control-plane
      - envoy-fleet
    volumes:
      - ./certs:/etc/certs

  # Envoy instance template (managed by fleet manager)
  # envoy-1:
  #   image: envoyproxy/envoy:v1.31-latest
  #   environment:
  #     NODE_ID: edge-001
  #     CONTROL_PLANE_HOST: control-plane
  #     CONTROL_PLANE_PORT: 18000
  #     CLUSTER_NAME: edge
  #   ports:
  #     - "10001:10000"
  #     - "20001:19000"
  #   networks:
  #     - envoy-fleet
  #     - backend-services

networks:
  control-plane:
    driver: bridge
  envoy-fleet:
    driver: bridge
  backend-services:
    driver: bridge

volumes:
  postgres_data:
```

### 7.2 Example Instance Configuration

```yaml
# Example configuration for instance edge-001
instance:
  node_id: edge-001
  cluster_name: edge
  metadata:
    datacenter: us-east-1
    environment: production
    role: edge-proxy

listeners:
  - name: http_listener
    address: 0.0.0.0
    port: 10000
    filter_chains:
      - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              stat_prefix: ingress_http
              route_config_name: main_routes
              http_filters:
                - name: envoy.filters.http.router

routes:
  - name: main_routes
    virtual_hosts:
      - name: api_host
        domains: ["api.example.com", "api.example.com:*"]
        routes:
          - match:
              prefix: "/v1/users"
            route:
              cluster: user_service
              timeout: 15s
          - match:
              prefix: "/v1/products"
            route:
              cluster: product_service
              timeout: 10s

clusters:
  - name: user_service
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: user_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: user-service
                    port_value: 8080
    health_checks:
      - timeout: 5s
        interval: 10s
        http_health_check:
          path: /health

  - name: product_service
    type: STRICT_DNS
    lb_policy: LEAST_REQUEST
    load_assignment:
      cluster_name: product_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: product-service
                    port_value: 8080
```

### 7.3 CLI Usage Examples

```bash
# Create a new fleet
envoy-fleet create --cluster edge --count 3 \
  --control-plane localhost:18000 \
  --network envoy-fleet

# List all instances
envoy-fleet list

# List instances in a cluster
envoy-fleet list --cluster edge

# Get instance details
envoy-fleet get edge-001

# Start an instance
envoy-fleet start edge-001

# Stop an instance
envoy-fleet stop edge-001

# Remove an instance
envoy-fleet remove edge-001

# Scale a cluster
envoy-fleet scale edge --count 5

# Apply configuration to instance
envoy-fleet config apply edge-001 --file config.yaml

# Get instance configuration
envoy-fleet config get edge-001

# View instance logs
envoy-fleet logs edge-001 --follow

# Check instance health
envoy-fleet health edge-001

# Get instance metrics
envoy-fleet metrics edge-001

# Restart all instances in a cluster
envoy-fleet restart --cluster edge

# Update all instances with new image
envoy-fleet update --cluster edge --image envoyproxy/envoy:v1.32-latest
```

---

## 8. Deployment and Operations

### 8.1 Initial Setup

```bash
# 1. Clone repository
git clone https://github.com/yourorg/envoy-control-plane.git
cd envoy-control-plane

# 2. Set up database
docker-compose up -d postgres redis
cargo install sqlx-cli
sqlx database create
sqlx migrate run

# 3. Build control plane
cargo build --release

# 4. Build Envoy Docker image
docker build -t envoy-fleet:latest -f Dockerfile.envoy .

# 5. Start control plane
./target/release/envoy-control-plane \
  --config config/control-plane.yaml \
  --grpc-port 18000 \
  --http-port 8080

# 6. Create Docker network
docker network create envoy-fleet

# 7. Deploy first fleet
envoy-fleet create --cluster edge --count 3
```

### 8.2 Configuration Management

```bash
# Create configuration template
envoy-fleet template create http-proxy \
  --file templates/http-proxy.yaml

# Apply template to instance
envoy-fleet config apply edge-001 \
  --template http-proxy \
  --param domain=api.example.com \
  --param backend_cluster=api_service

# Validate configuration before applying
envoy-fleet config validate --file config.yaml

# Roll back to previous configuration
envoy-fleet config rollback edge-001 --version v1.2.3

# Export configuration
envoy-fleet config export edge-001 --output edge-001-config.yaml

# Import configuration
envoy-fleet config import edge-002 --file edge-001-config.yaml
```

### 8.3 Monitoring and Troubleshooting

```bash
# Check cluster health
envoy-fleet health --cluster edge

# View real-time metrics
envoy-fleet metrics --cluster edge --watch

# Stream logs from multiple instances
envoy-fleet logs --cluster edge --follow --tail 100

# Get xDS connection status
envoy-fleet xds status edge-001

# View configuration history
envoy-fleet config history edge-001

# Debug configuration issues
envoy-fleet debug edge-001 --show-snapshot --show-ack-status
```

---

## 9. Testing Strategy

### 9.1 Unit Tests

```rust
// tests/fleet_manager_test.rs
#[tokio::test]
async fn test_create_instance() {
    let fleet_manager = setup_test_fleet_manager().await;

    let params = CreateInstanceParams {
        cluster_name: "test-cluster".to_string(),
        instance_number: 1,
        control_plane_host: "localhost".to_string(),
        control_plane_port: 18000,
        ..Default::default()
    };

    let instance = fleet_manager.create_instance(params).await.unwrap();

    assert_eq!(instance.cluster_name, "test-cluster");
    assert_eq!(instance.node_id, "test-cluster-001");
    assert!(instance.proxy_port > 0);
    assert!(instance.admin_port > 0);
}

#[tokio::test]
async fn test_scale_fleet() {
    let fleet_manager = setup_test_fleet_manager().await;

    // Scale up
    let instances = fleet_manager
        .scale_fleet("test-cluster", 5)
        .await
        .unwrap();

    assert_eq!(instances.len(), 5);

    // Scale down
    fleet_manager
        .scale_fleet("test-cluster", 2)
        .await
        .unwrap();

    let remaining = fleet_manager
        .list_instances(Some(InstanceFilter {
            cluster_name: Some("test-cluster".to_string()),
            status: Some(InstanceStatus::Running),
        }))
        .await
        .unwrap();

    assert_eq!(remaining.len(), 2);
}
```

### 9.2 Integration Tests

```rust
// tests/integration_test.rs
#[tokio::test]
async fn test_fleet_with_xds_server() {
    // Start control plane
    let control_plane = start_test_control_plane().await;

    // Create fleet manager
    let fleet_manager = FleetManager::new(/* ... */).await.unwrap();

    // Create instance
    let instance = fleet_manager.create_instance(/* ... */).await.unwrap();
    fleet_manager.start_instance(instance.id).await.unwrap();

    // Wait for xDS connection
    tokio::time::sleep(Duration::from_secs(2)).await;

    // Verify instance connected to control plane
    let status = control_plane.get_proxy_status(&instance.node_id).await.unwrap();
    assert_eq!(status.connection_state, "connected");

    // Apply configuration
    let config = create_test_config();
    control_plane.apply_config(&instance.node_id, config).await.unwrap();

    // Wait for configuration application
    tokio::time::sleep(Duration::from_secs(1)).await;

    // Verify configuration applied
    let applied_config = control_plane.get_applied_config(&instance.node_id).await.unwrap();
    assert!(applied_config.is_acknowledged);
}
```

### 9.3 Load Tests

```rust
// tests/load_test.rs
#[tokio::test]
#[ignore] // Run with: cargo test --ignored
async fn test_100_instance_fleet() {
    let fleet_manager = setup_test_fleet_manager().await;

    // Create 100 instances
    let start = Instant::now();
    let instances = fleet_manager.scale_fleet("load-test", 100).await.unwrap();
    let creation_time = start.elapsed();

    println!("Created 100 instances in {:?}", creation_time);
    assert!(creation_time < Duration::from_secs(60)); // Should complete in under 1 minute

    // Verify all instances are healthy
    tokio::time::sleep(Duration::from_secs(10)).await;

    for instance in &instances {
        let health = check_instance_health(&instance).await;
        assert_eq!(health, HealthStatus::Healthy);
    }

    // Apply configuration to all instances
    let config = create_test_config();
    let start = Instant::now();

    for instance in &instances {
        apply_config_to_instance(instance, &config).await.unwrap();
    }

    let config_time = start.elapsed();
    println!("Applied config to 100 instances in {:?}", config_time);
    assert!(config_time < Duration::from_secs(10)); // Should complete in under 10 seconds

    // Cleanup
    for instance in instances {
        fleet_manager.remove_instance(instance.id, true).await.unwrap();
    }
}
```

---

## 10. Monitoring and Observability

### 10.1 Metrics to Track

**Fleet-Level Metrics:**
- Total number of instances
- Instances by status (running, stopped, failed)
- Instances by cluster
- Total port utilization
- Container resource usage (CPU, memory)

**Instance-Level Metrics:**
- xDS connection status
- Configuration version
- Last configuration update timestamp
- Health check status
- Request throughput
- Error rates
- Response latencies (p50, p95, p99)

**Control Plane Metrics:**
- Configuration push rate
- Configuration ACK/NACK ratio
- xDS stream count
- Database query latency
- API request latency

### 10.2 Prometheus Metrics

```rust
// src/metrics.rs
use prometheus::{Registry, IntGauge, IntGaugeVec, HistogramVec, Opts};

pub struct FleetMetrics {
    pub total_instances: IntGauge,
    pub instances_by_status: IntGaugeVec,
    pub instances_by_cluster: IntGaugeVec,
    pub config_push_duration: HistogramVec,
    pub health_check_duration: HistogramVec,
}

impl FleetMetrics {
    pub fn new(registry: &Registry) -> Self {
        let total_instances = IntGauge::new(
            "envoy_fleet_total_instances",
            "Total number of Envoy instances"
        ).unwrap();

        let instances_by_status = IntGaugeVec::new(
            Opts::new("envoy_fleet_instances_by_status", "Instances grouped by status"),
            &["status"]
        ).unwrap();

        let instances_by_cluster = IntGaugeVec::new(
            Opts::new("envoy_fleet_instances_by_cluster", "Instances grouped by cluster"),
            &["cluster"]
        ).unwrap();

        let config_push_duration = HistogramVec::new(
            prometheus::HistogramOpts::new(
                "envoy_fleet_config_push_duration_seconds",
                "Time taken to push configuration to instance"
            ),
            &["instance_id", "result"]
        ).unwrap();

        let health_check_duration = HistogramVec::new(
            prometheus::HistogramOpts::new(
                "envoy_fleet_health_check_duration_seconds",
                "Time taken for health check"
            ),
            &["instance_id"]
        ).unwrap();

        registry.register(Box::new(total_instances.clone())).unwrap();
        registry.register(Box::new(instances_by_status.clone())).unwrap();
        registry.register(Box::new(instances_by_cluster.clone())).unwrap();
        registry.register(Box::new(config_push_duration.clone())).unwrap();
        registry.register(Box::new(health_check_duration.clone())).unwrap();

        Self {
            total_instances,
            instances_by_status,
            instances_by_cluster,
            config_push_duration,
            health_check_duration,
        }
    }
}
```

### 10.3 Logging Strategy

```rust
// Structured logging examples
tracing::info!(
    instance_id = %instance.id,
    node_id = %instance.node_id,
    cluster = %instance.cluster_name,
    proxy_port = instance.proxy_port,
    "Instance created successfully"
);

tracing::warn!(
    instance_id = %instance.id,
    health_status = ?instance.health_status,
    "Instance health check failed"
);

tracing::error!(
    instance_id = %instance.id,
    error = %error,
    "Failed to apply configuration"
);
```

### 10.4 Grafana Dashboard

```json
{
  "dashboard": {
    "title": "Envoy Fleet Overview",
    "panels": [
      {
        "title": "Total Instances",
        "targets": [
          {
            "expr": "envoy_fleet_total_instances"
          }
        ]
      },
      {
        "title": "Instances by Status",
        "targets": [
          {
            "expr": "envoy_fleet_instances_by_status"
          }
        ]
      },
      {
        "title": "Config Push Duration (p99)",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(envoy_fleet_config_push_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Health Check Success Rate",
        "targets": [
          {
            "expr": "sum(rate(envoy_fleet_health_checks_total{result=\"success\"}[5m])) / sum(rate(envoy_fleet_health_checks_total[5m]))"
          }
        ]
      }
    ]
  }
}
```

---

## Appendix A: Project Structure

```
envoy-control-plane/
├── Cargo.toml
├── Cargo.lock
├── Dockerfile
├── Dockerfile.envoy
├── docker-compose.yml
├── README.md
├── REQUIREMENTS.md
├── DESIGN.md
├── ENVOY_FLEET_IMPLEMENTATION.md
├── migrations/
│   ├── 001_create_instances.sql
│   ├── 002_create_configs.sql
│   └── 003_create_templates.sql
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── config/
│   │   ├── mod.rs
│   │   ├── assignment.rs
│   │   ├── manager.rs
│   │   ├── storage.rs
│   │   ├── templates.rs
│   │   └── validator.rs
│   ├── fleet/
│   │   ├── mod.rs
│   │   ├── manager.rs
│   │   ├── health.rs
│   │   └── ports.rs
│   ├── models/
│   │   ├── mod.rs
│   │   ├── instance.rs
│   │   ├── config.rs
│   │   └── snapshot.rs
│   ├── xds/
│   │   ├── mod.rs
│   │   ├── server.rs
│   │   ├── ads.rs
│   │   ├── lds.rs
│   │   ├── rds.rs
│   │   ├── cds.rs
│   │   ├── eds.rs
│   │   └── sds.rs
│   ├── api/
│   │   ├── mod.rs
│   │   ├── rest.rs
│   │   └── grpc.rs
│   ├── cli/
│   │   ├── mod.rs
│   │   ├── main.rs
│   │   └── commands.rs
│   └── metrics/
│       ├── mod.rs
│       └── collector.rs
├── templates/
│   ├── http_listener.hbs
│   ├── tcp_listener.hbs
│   ├── http_route.hbs
│   └── cluster.hbs
├── envoy/
│   ├── bootstrap-template.yaml
│   └── entrypoint.sh
├── config/
│   ├── control-plane.yaml
│   └── examples/
│       ├── edge-proxy.yaml
│       ├── internal-proxy.yaml
│       └── api-gateway.yaml
└── tests/
    ├── unit/
    │   ├── fleet_manager_test.rs
    │   ├── config_manager_test.rs
    │   └── health_monitor_test.rs
    ├── integration/
    │   ├── fleet_lifecycle_test.rs
    │   └── xds_integration_test.rs
    └── load/
        └── fleet_load_test.rs
```

---

## Appendix B: Dependencies

```toml
# Cargo.toml
[package]
name = "envoy-control-plane"
version = "0.1.0"
edition = "2021"

[dependencies]
# Docker client
bollard = "0.16"

# Async runtime
tokio = { version = "1.35", features = ["full"] }
tokio-stream = "0.1"

# gRPC
tonic = "0.11"
prost = "0.12"

# Database
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-native-tls", "uuid", "chrono", "json"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serde_yaml = "0.9"

# Template engine
handlebars = "5.1"

# HTTP client
reqwest = { version = "0.11", features = ["json"] }

# CLI
clap = { version = "4.4", features = ["derive"] }

# Metrics
prometheus = "0.13"

# Logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# Error handling
anyhow = "1.0"
thiserror = "1.0"

# Utilities
uuid = { version = "1.6", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }

[dev-dependencies]
testcontainers = "0.15"
```

---

**End of Implementation Plan**
