# Containerization and Orchestration

<cite>
**Referenced Files in This Document**
- [Dockerfile](file://Dockerfile)
- [compose.yaml](file://compose.yaml)
- [dev-compose.yaml](file://dev-compose.yaml)
- [development.compose.yml](file://development.compose.yml)
- [docker-compose-vllm.yaml](file://docker-compose-vllm.yaml)
- [scripts/docker-compose.yml](file://scripts/docker-compose.yml)
- [scripts/pack_docker.sh](file://scripts/pack_docker.sh)
- [scripts/wait_for_service.sh](file://scripts/wait_for_service.sh)
- [otel/otel-collector-config-file.yaml](file://otel/otel-collector-config-file.yaml)
- [otel/otel-collector-config-clickhouse.yaml](file://otel/otel-collector-config-clickhouse.yaml)
- [otel/otel-collector-config-signoz.yaml](file://otel/otel-collector-config-signoz.yaml)
- [otel/start-otel-collector.sh](file://otel/start-otel-collector.sh)
- [letta/server/startup.sh](file://letta/server/startup.sh)
- [nginx.conf](file://nginx.conf)
- [init.sql](file://init.sql)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Docker Image Architecture](#docker-image-architecture)
3. [OpenTelemetry Collector Configuration](#opentelemetry-collector-configuration)
4. [Docker Compose Orchestration](#docker-compose-orchestration)
5. [Production vs Development Topologies](#production-vs-development-topologies)
6. [Telemetry Data Flow](#telemetry-data-flow)
7. [Best Practices](#best-practices)
8. [Integration Points](#integration-points)
9. [Monitoring and Health Checks](#monitoring-and-health-checks)
10. [Conclusion](#conclusion)

## Introduction

The Letta project implements a comprehensive containerization and orchestration strategy centered around the OpenTelemetry (OTEL) collector for distributed tracing, metrics, and logging. This architecture enables scalable deployment across development and production environments while maintaining observability and operational excellence.

The containerization strategy employs multi-stage Docker builds, sophisticated orchestration through Docker Compose, and flexible configuration management for different deployment scenarios. The OpenTelemetry collector serves as the central telemetry hub, supporting multiple export destinations including file-based storage, ClickHouse analytics, and external monitoring platforms like Grafana and Signoz.

## Docker Image Architecture

### Multi-Stage Build Process

The Docker image follows a sophisticated multi-stage architecture designed for optimal size and security:

```mermaid
flowchart TD
A["Base Image<br/>ankane/pgvector:v0.5.1"] --> B["Builder Stage"]
B --> C["Python Environment Setup"]
C --> D["Dependencies Installation"]
D --> E["Application Build"]
E --> F["Runtime Stage"]
F --> G["Final Image"]
H["OpenTelemetry Collector<br/>v0.96.0"] --> F
I["Node.js v22"] --> F
J["PostgreSQL Client Libraries"] --> F
K["Configuration Files"] --> F
L["Health Check Scripts"] --> F
M["Startup Scripts"] --> F
```

**Diagram sources**
- [Dockerfile](file://Dockerfile#L1-L89)

### Base Image Selection

The architecture utilizes `ankane/pgvector:v0.5.1` as the foundation, providing:
- Pre-configured PostgreSQL with vector extensions
- Optimized for machine learning workloads
- Production-ready database capabilities
- Vector search functionality for semantic storage

### Binary Installation Strategy

The OpenTelemetry collector is installed using a sophisticated download and extraction process:

```mermaid
sequenceDiagram
participant Script as "Installation Script"
participant GitHub as "GitHub Releases"
participant Local as "Local System"
Script->>Script : Detect Platform (OS + Arch)
Script->>GitHub : Download otelcol-contrib_<version>_<platform>.tar.gz
GitHub-->>Script : Binary Archive
Script->>Local : Extract to /usr/local/bin
Script->>Local : Set executable permissions
Script->>Local : Configure environment variables
```

**Diagram sources**
- [otel/start-otel-collector.sh](file://otel/start-otel-collector.sh#L98-L120)

### Entrypoint Configuration

The container uses a dual-entrypoint strategy combining PostgreSQL and Letta server startup:

**Section sources**
- [Dockerfile](file://Dockerfile#L87-L89)
- [letta/server/startup.sh](file://letta/server/startup.sh#L1-L82)

## OpenTelemetry Collector Configuration

### Configuration Architecture

The OpenTelemetry collector supports multiple deployment scenarios through environment-specific configurations:

```mermaid
graph TB
subgraph "Collector Configurations"
A["File Export<br/>otel-collector-config-file.yaml"]
B["ClickHouse Export<br/>otel-collector-config-clickhouse.yaml"]
C["Signoz Export<br/>otel-collector-config-signoz.yaml"]
end
subgraph "Export Destinations"
D["Local Files<br/>/root/.letta/logs/"]
E["ClickHouse Database<br/>Analytics Pipeline"]
F["Signoz Monitoring<br/>External Platform"]
end
subgraph "Data Sources"
G["OTLP Protocol<br/>GRPC/HTTP"]
H["Log Files<br/>/root/.letta/logs/"]
end
G --> A
G --> B
G --> C
H --> B
H --> C
A --> D
B --> E
C --> F
```

**Diagram sources**
- [otel/otel-collector-config-file.yaml](file://otel/otel-collector-config-file.yaml#L1-L31)
- [otel/otel-collector-config-clickhouse.yaml](file://otel/otel-collector-config-clickhouse.yaml#L1-L82)
- [otel/otel-collector-config-signoz.yaml](file://otel/otel-collector-config-signoz.yaml#L1-L49)

### File-Based Configuration

The default configuration focuses on local file export with rotation capabilities:

| Component | Configuration | Purpose |
|-----------|---------------|---------|
| **Receivers** | OTLP (GRPC:4317, HTTP:4318) | Collect telemetry data from applications |
| **Processors** | Batch (timeout: 1s, size: 1024) | Optimize throughput with batching |
| **Exporters** | File (rotation: 100MB, 7 days, 5 backups) | Persistent local storage |
| **Service** | Logs level: error | Minimal logging overhead |

### ClickHouse Analytics Configuration

Production deployments utilize ClickHouse for centralized analytics:

| Feature | Configuration | Benefits |
|---------|---------------|----------|
| **Memory Management** | Limit: 1024MiB, Spike: 256MiB | Prevent memory exhaustion |
| **Batch Processing** | Timeout: 10s, Size: 8192 | Optimal throughput balancing |
| **Retry Logic** | Initial: 5s, Max: 30s, Total: 300s | Handle temporary failures |
| **Queue Management** | Size: 500, Enabled | Buffer during connectivity issues |

### Environment Variable Integration

The collector dynamically selects configurations based on environment variables:

```mermaid
flowchart TD
A["Environment Check"] --> B{"CLICKHOUSE_ENDPOINT<br/>+ CLICKHOUSE_PASSWORD?"}
B --> |Yes| C["ClickHouse Configuration"]
B --> |No| D{"SIGNOZ_ENDPOINT<br/>+ SIGNOZ_INGESTION_KEY?"}
D --> |Yes| E["Signoz Configuration"]
D --> |No| F["File Configuration"]
C --> G["Analytics Pipeline"]
E --> H["External Monitoring"]
F --> I["Local Storage"]
```

**Diagram sources**
- [otel/start-otel-collector.sh](file://otel/start-otel-collector.sh#L133-L139)
- [letta/server/startup.sh](file://letta/server/startup.sh#L57-L66)

**Section sources**
- [otel/otel-collector-config-file.yaml](file://otel/otel-collector-config-file.yaml#L1-L31)
- [otel/otel-collector-config-clickhouse.yaml](file://otel/otel-collector-config-clickhouse.yaml#L1-L82)
- [otel/otel-collector-config-signoz.yaml](file://otel/otel-collector-config-signoz.yaml#L1-L49)

## Docker Compose Orchestration

### Production Deployment Architecture

The production deployment orchestrates multiple services with careful dependency management:

```mermaid
graph TB
subgraph "Network Layer"
A["Nginx Reverse Proxy<br/>Port 80"]
B["Letta Server<br/>Port 8083, 8283"]
C["PostgreSQL Database<br/>Port 5432"]
end
subgraph "Service Dependencies"
D["letta_nginx"] --> B
B --> C
B --> E["OpenTelemetry Collector<br/>Port 4317, 4318"]
end
subgraph "Volume Management"
F[".persist/pgdata"]
G["init.sql"]
H["nginx.conf"]
end
C -.-> F
C -.-> G
A -.-> H
```

**Diagram sources**
- [compose.yaml](file://compose.yaml#L1-L66)

### Development Environment Setup

The development environment emphasizes rapid iteration and debugging capabilities:

```mermaid
sequenceDiagram
participant Dev as "Developer"
participant Docker as "Docker Engine"
participant Builder as "Build Stage"
participant Runtime as "Runtime Stage"
Dev->>Docker : docker-compose build
Docker->>Builder : Copy source code
Docker->>Builder : Install dependencies
Docker->>Builder : Build application
Docker->>Runtime : Copy built artifacts
Docker->>Runtime : Install runtime dependencies
Runtime-->>Dev : Ready for development
```

**Diagram sources**
- [dev-compose.yaml](file://dev-compose.yaml#L19-L24)
- [development.compose.yml](file://development.compose.yml#L5-L8)

### Volume Mount Strategies

Different environments employ distinct volume mounting approaches:

| Environment | Database Volume | Application Volume | Configuration Volume |
|-------------|-----------------|-------------------|---------------------|
| **Production** | Persistent `.persist/pgdata` | None (immutable) | Environment-specific configs |
| **Development** | Temporary `/var/lib/postgresql/data` | Source code bind mount | Config files in container |
| **Testing** | Test-specific data dir | None | Test configurations |

### Network Isolation

Services are organized into logical networks with alias support:

```mermaid
graph LR
subgraph "Default Network"
A["letta-db<br/>Aliases: pgvector_db, letta-db"]
B["letta-server"]
C["letta-nginx"]
end
D["External Access"] --> C
B --> A
C --> B
```

**Diagram sources**
- [compose.yaml](file://compose.yaml#L2-L8)

**Section sources**
- [compose.yaml](file://compose.yaml#L1-L66)
- [dev-compose.yaml](file://dev-compose.yaml#L1-L49)
- [development.compose.yml](file://development.compose.yml#L1-L30)

## Production vs Development Topologies

### Scaling Considerations

The architecture supports different scaling patterns for various deployment scenarios:

```mermaid
graph TB
subgraph "Development Topology"
A["Single Instance<br/>letta_server"]
B["Embedded PostgreSQL<br/>letta_db"]
C["Local Nginx<br/>letta_nginx"]
end
subgraph "Production Topology"
D["Load Balancer"]
E["Multiple Letta Servers<br/>Horizontal Scaling"]
F["Database Cluster<br/>High Availability"]
G["Reverse Proxy Cluster<br/>Nginx Load Balancing"]
end
A --> B
A --> C
D --> E
E --> F
D --> G
```

### Resource Management

Production deployments implement comprehensive resource controls:

| Resource Type | Development | Production | Rationale |
|---------------|-------------|------------|-----------|
| **CPU Limits** | No limits | CPU requests/limits | Prevent resource contention |
| **Memory Limits** | No limits | Memory requests/limits | Ensure predictable performance |
| **Storage** | Local SSD | Network storage | Data durability and backup |
| **Network** | Bridge networking | Overlay networking | Service discovery and isolation |

### Security Posture Differences

Security implementations vary significantly between environments:

```mermaid
flowchart TD
A["Security Comparison"] --> B["Development"]
A --> C["Production"]
B --> D["Minimal TLS<br/>Local network only"]
B --> E["Debug mode enabled<br/>Verbose logging"]
B --> F["Shared secrets<br/>Development keys"]
C --> G["Mutual TLS<br/>Network encryption"]
C --> H["Audit logging<br/>Security monitoring"]
C --> I["Secret management<br/>Vault/Kubernetes secrets"]
```

**Section sources**
- [compose.yaml](file://compose.yaml#L24-L66)
- [dev-compose.yaml](file://dev-compose.yaml#L19-L49)

## Telemetry Data Flow

### Distributed Tracing Architecture

The telemetry system implements comprehensive distributed tracing across microservices:

```mermaid
sequenceDiagram
participant Client as "Client Application"
participant Nginx as "Nginx Proxy"
participant Server as "Letta Server"
participant Collector as "OTEL Collector"
participant Exporter as "Export Destination"
Client->>Nginx : HTTP Request
Nginx->>Server : Forward Request
Server->>Server : Process Business Logic
Server->>Collector : Send OTLP Traces
Collector->>Exporter : Export to Destination
Collector-->>Server : Acknowledge
Server-->>Nginx : HTTP Response
Nginx-->>Client : Final Response
```

**Diagram sources**
- [nginx.conf](file://nginx.conf#L1-L29)
- [letta/server/startup.sh](file://letta/server/startup.sh#L57-L66)

### Metrics Collection Pipeline

The system captures comprehensive metrics across multiple dimensions:

| Metric Category | Data Source | Collection Method | Export Destination |
|-----------------|-------------|-------------------|-------------------|
| **Application Metrics** | Letta Server | OTLP HTTP/GRPC | ClickHouse/Signoz |
| **Infrastructure Metrics** | Docker Engine | Prometheus exporter | Central monitoring |
| **Database Metrics** | PostgreSQL | pg_stat_* views | ClickHouse |
| **System Metrics** | Host OS | Node exporter | Central monitoring |

### Log Aggregation Strategy

Multi-source log aggregation ensures comprehensive observability:

```mermaid
flowchart TD
A["Log Sources"] --> B["File Log Receiver"]
C["Application Logs"] --> B
D["System Logs"] --> B
E["Container Logs"] --> B
B --> F["Log Processing"]
F --> G["JSON Parsing"]
F --> H["Timestamp Extraction"]
F --> I["Attribute Enrichment"]
G --> J["ClickHouse Export"]
H --> J
I --> J
```

**Diagram sources**
- [otel/otel-collector-config-clickhouse.yaml](file://otel/otel-collector-config-clickhouse.yaml#L8-L25)

**Section sources**
- [letta/server/startup.sh](file://letta/server/startup.sh#L57-L66)
- [otel/otel-collector-config-clickhouse.yaml](file://otel/otel-collector-config-clickhouse.yaml#L8-L25)

## Best Practices

### Image Versioning Strategy

The containerization implements robust versioning practices:

```mermaid
flowchart LR
A["Version Detection"] --> B["Semantic Versioning"]
B --> C["Multi-Architecture Builds"]
C --> D["Immutable Tags"]
D --> E["Registry Push"]
F["scripts/pack_docker.sh"] --> A
G["letta version"] --> B
H["docker buildx"] --> C
I["letta/letta-server:${VERSION}"] --> D
```

**Diagram sources**
- [scripts/pack_docker.sh](file://scripts/pack_docker.sh#L1-L4)

### Health Check Implementation

Comprehensive health checking ensures service reliability:

| Service | Health Check Type | Endpoint | Frequency | Timeout |
|---------|------------------|----------|-----------|---------|
| **PostgreSQL** | Command shell | `pg_isready` | 5s | 5s |
| **Letta Server** | HTTP endpoint | `/health` | 10s | 3s |
| **OpenTelemetry Collector** | Built-in | `:8888/health` | 15s | 5s |
| **Nginx** | TCP connection | Port 80 | 1s | 3s |

### Rolling Update Strategy

The architecture supports zero-downtime deployments:

```mermaid
sequenceDiagram
participant LB as "Load Balancer"
participant Old as "Old Instance"
participant New as "New Instance"
participant DB as "Database"
LB->>New : Health check (passing)
LB->>New : Start accepting traffic
LB->>Old : Remove from rotation
Old->>DB : Graceful shutdown
New->>DB : Take over connections
LB->>Old : Terminate gracefully
```

### Resource Limits and Monitoring

Production deployments implement strict resource controls:

| Resource | Development | Production | Purpose |
|----------|-------------|------------|---------|
| **CPU** | Unlimited | Requests: 500m, Limits: 2000m | Prevent CPU starvation |
| **Memory** | Unlimited | Requests: 1Gi, Limits: 4Gi | Ensure predictable allocation |
| **Storage** | Local | Persistent volumes | Data durability |
| **Network** | Bridge | Overlay networks | Service isolation |

**Section sources**
- [scripts/pack_docker.sh](file://scripts/pack_docker.sh#L1-L4)
- [compose.yaml](file://compose.yaml#L18-L22)
- [dev-compose.yaml](file://dev-compose.yaml#L17-L18)

## Integration Points

### Main Application Services

The OpenTelemetry collector integrates seamlessly with Letta's core services:

```mermaid
graph TB
subgraph "Letta Core Services"
A["Agent Manager"]
B["Message Processor"]
C["Tool Executor"]
D["Memory Manager"]
end
subgraph "Telemetry Integration"
E["OTLP Exporter"]
F["Trace Context Propagation"]
G["Metric Collection"]
end
subgraph "External Systems"
H["ClickHouse Analytics"]
I["Grafana Dashboard"]
J["Signoz Monitoring"]
end
A --> E
B --> E
C --> E
D --> E
E --> F
F --> G
G --> H
G --> I
G --> J
```

### Service Discovery Mechanisms

The architecture supports multiple service discovery approaches:

| Discovery Method | Use Case | Configuration | Benefits |
|------------------|----------|---------------|----------|
| **DNS-based** | Internal services | Container aliases | Simple, reliable |
| **Environment vars** | External services | `LETTA_PG_URI` | Flexible configuration |
| **File-based** | Secrets management | `/etc/otel/*` | Secure credential handling |

### API Gateway Integration

Nginx serves as the primary API gateway with sophisticated routing:

```mermaid
flowchart TD
A["Incoming Request"] --> B{"Path Matching"}
B --> |/api/*| C["REST API Routes"]
B --> |WebSocket| D["WebSocket Upgrades"]
B --> |Static| E["Static Content"]
C --> F["letta-server:8283"]
D --> F
E --> G["Static File Serving"]
F --> H["Response"]
G --> H
```

**Diagram sources**
- [nginx.conf](file://nginx.conf#L1-L29)

**Section sources**
- [nginx.conf](file://nginx.conf#L1-L29)
- [letta/server/startup.sh](file://letta/server/startup.sh#L57-L66)

## Monitoring and Health Checks

### Comprehensive Health Monitoring

The architecture implements multi-layered health monitoring:

```mermaid
graph TB
subgraph "Application Layer"
A["Letta Server Health"]
B["Database Connectivity"]
C["OpenTelemetry Collector"]
end
subgraph "Infrastructure Layer"
D["PostgreSQL Health"]
E["Nginx Health"]
F["System Resources"]
end
subgraph "External Monitoring"
G["Grafana Dashboards"]
H["Alert Manager"]
I["Central Logging"]
end
A --> B
A --> C
B --> D
C --> E
D --> F
A --> G
B --> G
C --> G
G --> H
G --> I
```

### Performance Metrics

Key performance indicators are tracked across the stack:

| Metric Category | Measurement | Threshold | Action |
|-----------------|-------------|-----------|--------|
| **Response Time** | p95 latency | < 200ms | Alert on degradation |
| **Throughput** | Requests per second | Baseline ± 20% | Scale up/down |
| **Error Rate** | HTTP 5xx percentage | < 1% | Immediate investigation |
| **Resource Utilization** | CPU/Memory usage | > 80% | Capacity planning |

### Alerting Strategy

The monitoring system implements intelligent alerting:

```mermaid
flowchart TD
A["Metric Collection"] --> B{"Threshold Check"}
B --> |Above threshold| C["Alert Triggered"]
B --> |Normal range| D["Continue Monitoring"]
C --> E["Notification Channel"]
E --> F["Slack/Teams"]
E --> G["PagerDuty"]
E --> H["Email"]
C --> I["Runbook Execution"]
I --> J["Automated Remediation"]
I --> K["Manual Intervention"]
```

**Section sources**
- [compose.yaml](file://compose.yaml#L18-L22)
- [dev-compose.yaml](file://dev-compose.yaml#L17-L18)

## Conclusion

The Letta project's containerization and orchestration strategy demonstrates enterprise-grade observability and deployment practices. The multi-stage Docker build process ensures optimal image size and security, while the sophisticated OpenTelemetry collector configuration supports diverse deployment scenarios from local development to production analytics.

The architecture's strength lies in its flexibility and scalability, enabling seamless transitions between development and production environments while maintaining comprehensive observability. The integration of multiple export destinations (file, ClickHouse, Signoz) provides organizations with the choice of monitoring solutions that best fit their infrastructure and compliance requirements.

Key architectural benefits include:

- **Scalability**: Horizontal scaling support through load balancers and database clustering
- **Observability**: Comprehensive tracing, metrics, and logging across all services
- **Flexibility**: Multiple deployment configurations for different environments
- **Maintainability**: Automated health checks, rolling updates, and comprehensive monitoring
- **Security**: Environment-specific security postures with proper isolation

This containerization strategy positions Letta as a production-ready platform capable of handling enterprise-scale deployments while maintaining developer productivity and operational excellence.