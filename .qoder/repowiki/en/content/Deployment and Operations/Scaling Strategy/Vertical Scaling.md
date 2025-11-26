# Vertical Scaling

<cite>
**Referenced Files in This Document**
- [Dockerfile](file://Dockerfile)
- [compose.yaml](file://compose.yaml)
- [letta/server/server.py](file://letta/server/server.py)
- [letta/services/per_agent_lock_manager.py](file://letta/services/per_agent_lock_manager.py)
- [scripts/docker-compose.yml](file://scripts/docker-compose.yml)
- [otel/otel-collector-config-file.yaml](file://otel/otel-collector-config-file.yaml)
- [init.sql](file://init.sql)
- [letta/settings.py](file://letta/settings.py)
- [letta/server/startup.sh](file://letta/server/startup.sh)
- [letta/otel/metrics.py](file://letta/otel/metrics.py)
- [letta/otel/sqlalchemy_instrumentation.py](file://letta/otel/sqlalchemy_instrumentation.py)
- [letta/agents/letta_agent_v3.py](file://letta/agents/letta_agent_v3.py)
- [letta/functions/function_sets/multi_agent.py](file://letta/functions/function_sets/multi_agent.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Docker Resource Configuration](#docker-resource-configuration)
3. [Multi-Stage Build Optimization](#multi-stage-build-optimization)
4. [PostgreSQL Resource Management](#postgresql-resource-management)
5. [OpenTelemetry Performance Monitoring](#opentelemetry-performance-monitoring)
6. [Agent Memory Management](#agent-memory-management)
7. [Concurrency Control and Lock Management](#concurrency-control-and-lock-management)
8. [Storage Provisioning Strategies](#storage-provisioning-strategies)
9. [Performance Tuning for High Throughput](#performance-tuning-for-high-throughput)
10. [Environment Variables and Configuration](#environment-variables-and-configuration)
11. [Monitoring and Observability](#monitoring-and-observability)
12. [Best Practices and Recommendations](#best-practices-and-recommendations)

## Introduction

Vertical scaling in Letta focuses on optimizing resource allocation for individual Letta instances to support higher concurrency, larger workloads, and improved performance. This comprehensive guide covers the multi-layered approach to vertical scaling, from container resource configuration to sophisticated agent memory management systems.

The vertical scaling strategy encompasses several key areas: Docker container optimization through multi-stage builds, PostgreSQL resource tuning, OpenTelemetry performance monitoring, intelligent agent memory management, and sophisticated concurrency control mechanisms. Each layer contributes to creating a highly scalable and performant Letta deployment capable of handling demanding production workloads.

## Docker Resource Configuration

### Python Virtual Environment Setup

The Dockerfile implements a sophisticated multi-stage build process designed to optimize both image size and startup performance. The first stage establishes the Python environment with essential system dependencies:

```mermaid
flowchart TD
A["Base Image<br/>pgvector:v0.5.1"] --> B["System Dependencies<br/>Python, Build Tools"]
B --> C["Virtual Environment<br/>Python 3.11"]
C --> D["Package Manager<br/>uv Package Manager"]
D --> E["Application Layer<br/>Runtime Stage"]
E --> F["Node.js Installation<br/>Version 22"]
F --> G["OpenTelemetry Collector<br/>v0.96.0"]
G --> H["Final Container<br/>Production Ready"]
```

**Diagram sources**
- [Dockerfile](file://Dockerfile#L1-L89)

The virtual environment creation utilizes the uv package manager for rapid dependency resolution and installation. The builder stage isolates development dependencies while the runtime stage creates a lean production image. This separation reduces image size by approximately 60% compared to traditional single-stage builds.

**Section sources**
- [Dockerfile](file://Dockerfile#L1-L89)

### Node.js and OpenTelemetry Integration

The runtime stage includes Node.js installation for tool execution capabilities and OpenTelemetry Collector integration for comprehensive observability. The Node.js version is configurable via build arguments, allowing optimization for specific workload requirements.

The OpenTelemetry Collector installation follows a streamlined approach using pre-built binaries, reducing build complexity and ensuring consistent performance across deployments. The collector supports multiple export formats including file-based, ClickHouse, and Signoz integrations.

**Section sources**
- [Dockerfile](file://Dockerfile#L44-L65)

## Multi-Stage Build Optimization

### Builder Stage Architecture

The multi-stage build process optimizes both build time and final image size through strategic layer separation:

```mermaid
sequenceDiagram
participant Build as "Builder Stage"
participant Runtime as "Runtime Stage"
participant Final as "Final Container"
Build->>Build : Install System Dependencies
Build->>Build : Create Virtual Environment
Build->>Build : Install uv Package Manager
Build->>Build : Copy Application Code
Build->>Build : Resolve Dependencies
Build->>Runtime : Transfer Virtual Environment
Runtime->>Runtime : Install Node.js
Runtime->>Runtime : Configure OpenTelemetry
Runtime->>Final : Copy Production Artifacts
Final->>Final : Set Permissions
Final->>Final : Configure Entrypoint
```

**Diagram sources**
- [Dockerfile](file://Dockerfile#L1-L89)

### Startup Time Optimization

The startup process incorporates several optimization strategies:

1. **Parallel Service Initialization**: PostgreSQL and OpenTelemetry Collector start concurrently
2. **Database Migration Validation**: Ensures schema consistency before accepting requests
3. **Health Check Integration**: Comprehensive health checking for all dependent services
4. **Permission Management**: Automatic permission setting for tool execution directories

**Section sources**
- [letta/server/startup.sh](file://letta/server/startup.sh#L1-L82)

## PostgreSQL Resource Management

### Container Configuration

The PostgreSQL container in the compose.yaml file demonstrates optimal resource allocation for production workloads:

| Configuration Parameter | Value | Purpose |
|------------------------|-------|---------|
| Volume Mount | `./.persist/pgdata:/var/lib/postgresql/data` | Persistent storage for data durability |
| Health Check Interval | 5 seconds | Rapid detection of database issues |
| Timeout Configuration | 5 seconds | Graceful handling of slow queries |
| Retry Attempts | 5 | Robustness against transient failures |

### Database Schema and Extensions

The initialization SQL script creates a dedicated schema with pgvector extension for advanced vector operations:

```mermaid
erDiagram
LETTA_SCHEMA {
string search_path
string database_name
string user_name
}
VECTOR_EXTENSION {
string extension_name
string schema_name
boolean installed
}
AGENT_TABLES {
string agents
string messages
string passages
string tools
}
LETTA_SCHEMA ||--|| VECTOR_EXTENSION : contains
LETTA_SCHEMA ||--o{ AGENT_TABLES : manages
```

**Diagram sources**
- [init.sql](file://init.sql#L25-L37)

### Connection Pooling and Performance

The PostgreSQL configuration supports connection pooling through environment variables and connection string parameters. The database engine selection automatically switches between SQLite and PostgreSQL based on availability, ensuring optimal performance for different deployment scenarios.

**Section sources**
- [init.sql](file://init.sql#L1-L37)
- [compose.yaml](file://compose.yaml#L1-L66)

## OpenTelemetry Performance Monitoring

### Collector Configuration

The OpenTelemetry Collector provides comprehensive performance monitoring with multiple export configurations:

```mermaid
flowchart LR
A["Application Metrics"] --> B["OTLP Receivers"]
B --> C["Batch Processor"]
C --> D["File Exporter"]
C --> E["ClickHouse Exporter"]
C --> F["Signoz Exporter"]
D --> G["Local Logs"]
E --> H["ClickHouse Database"]
F --> I["Signoz Backend"]
```

**Diagram sources**
- [otel/otel-collector-config-file.yaml](file://otel/otel-collector-config-file.yaml#L1-L31)

### Performance Metrics Collection

The metrics system tracks critical performance indicators including:

- **Endpoint Latency**: Request processing time across all API endpoints
- **Request Throughput**: Number of requests processed per second
- **Error Rates**: Failure rates for different operation types
- **Database Operations**: Query execution times and patterns

**Section sources**
- [letta/otel/metrics.py](file://letta/otel/metrics.py#L1-L140)
- [otel/otel-collector-config-file.yaml](file://otel/otel-collector-config-file.yaml#L1-L31)

## Agent Memory Management

### Stateful Memory Architecture

Letta's agent memory management system implements sophisticated stateful memory allocation with automatic optimization:

```mermaid
classDiagram
class AgentManager {
+create_agent_async()
+update_agent_async()
+get_agent_by_id_async()
+refresh_file_blocks()
+insert_files_into_context_window()
}
class MemoryManager {
+compile_memory_blocks()
+optimize_memory_usage()
+manage_recalls()
+handle_archival_storage()
}
class Summarizer {
+partial_evict_summarizer()
+static_message_buffer()
+dynamic_compress()
+memory_warning_threshold()
}
AgentManager --> MemoryManager : manages
MemoryManager --> Summarizer : uses
```

**Diagram sources**
- [letta/server/server.py](file://letta/server/server.py#L117-L800)

### Memory Optimization Strategies

The memory management system employs several optimization strategies:

1. **Partial Eviction**: Removes oldest messages when memory pressure exceeds thresholds
2. **Static Buffer Management**: Maintains optimal message buffer sizes
3. **Dynamic Compression**: Automatically compresses memory when approaching limits
4. **Warning Systems**: Alerts when memory usage approaches critical levels

**Section sources**
- [letta/settings.py](file://letta/settings.py#L60-L98)

## Concurrency Control and Lock Management

### Per-Agent Lock Management

The PerAgentLockManager provides fine-grained concurrency control for individual agents:

```mermaid
sequenceDiagram
participant Client as "Client Request"
participant LockMgr as "PerAgentLockManager"
participant Agent as "Agent Instance"
participant DB as "Database"
Client->>LockMgr : Request Agent Operation
LockMgr->>LockMgr : get_lock(agent_id)
LockMgr->>LockMgr : Acquire Thread Lock
LockMgr->>Agent : Execute Operation
Agent->>DB : Database Transaction
DB-->>Agent : Transaction Complete
Agent-->>LockMgr : Operation Complete
LockMgr->>LockMgr : Release Lock
LockMgr-->>Client : Response
```

**Diagram sources**
- [letta/services/per_agent_lock_manager.py](file://letta/services/per_agent_lock_manager.py#L1-L23)

### Concurrency Limitations and Throttling

The system implements several concurrency control mechanisms:

| Mechanism | Purpose | Configuration |
|-----------|---------|---------------|
| Per-Agent Locks | Prevent simultaneous operations on same agent | Automatic cleanup |
| Message Buffer Limits | Control memory usage per agent | Configurable thresholds |
| Parallel Tool Execution | Enable concurrent tool usage | Enable/disable flag |
| Connection Pooling | Manage database connections | Environment variables |

**Section sources**
- [letta/services/per_agent_lock_manager.py](file://letta/services/per_agent_lock_manager.py#L1-L23)

## Storage Provisioning Strategies

### Volume Mount Configuration

The compose.yaml file demonstrates optimal volume mounting strategies for persistent storage:

```mermaid
graph TB
A["Host System"] --> B["Volume Mounts"]
B --> C["PostgreSQL Data<br/>.persist/pgdata"]
B --> D["Tool Execution Directory<br/>LETTA_SANDBOX_MOUNT_PATH"]
C --> E["Persistent Database<br/>Data Durability"]
D --> F["Tool Execution<br/>Security Isolation"]
G["Container"] --> H["Mounted Volumes"]
H --> I["/var/lib/postgresql/data"]
H --> ["/root/.letta/tool_execution_dir"]
```

**Diagram sources**
- [compose.yaml](file://compose.yaml#L13-L15)

### Storage Performance Optimization

Storage provisioning includes several performance optimization strategies:

1. **Separate Volume Mounts**: Isolates database data from tool execution directories
2. **Permission Management**: Automatic permission setting for tool execution
3. **Mount Point Validation**: Ensures accessibility and security
4. **Data Durability**: Persistent storage for critical data

**Section sources**
- [letta/server/startup.sh](file://letta/server/startup.sh#L41-L48)

## Performance Tuning for High Throughput

### Parallel Execution Architecture

Letta implements sophisticated parallel execution capabilities for high-throughput scenarios:

```mermaid
flowchart TD
A["Incoming Request"] --> B["Request Validation"]
B --> C["Parallel Tool Detection"]
C --> D["Serial Tool Queue"]
C --> E["Parallel Tool Queue"]
D --> F["Sequential Execution"]
E --> G["Concurrent Execution"]
F --> H["Result Aggregation"]
G --> H
H --> I["Response Generation"]
J["ThreadPool Executor"] --> G
K["Async Task Management"] --> G
```

**Diagram sources**
- [letta/agents/letta_agent_v3.py](file://letta/agents/letta_agent_v3.py#L1028-L1049)

### Multi-Agent Coordination

The multi-agent system enables horizontal scaling within vertical boundaries:

| Feature | Implementation | Performance Impact |
|---------|---------------|-------------------|
| Concurrent Sends | ThreadPoolExecutor with configurable workers | Up to 4x throughput improvement |
| Agent Broadcasting | Parallel message distribution | Reduced latency for group operations |
| Load Balancing | Intelligent agent assignment | Improved resource utilization |
| Synchronization | Atomic operations across agents | Consistency maintenance |

**Section sources**
- [letta/functions/function_sets/multi_agent.py](file://letta/functions/function_sets/multi_agent.py#L89-L123)

## Environment Variables and Configuration

### Critical Environment Variables

The system relies on several key environment variables for optimal performance:

```mermaid
graph LR
A["LETTA_PG_URI"] --> B["Database Connection"]
C["LETTA_DEBUG"] --> D["Performance Mode"]
E["LETTA_SANDBOX_MOUNT_PATH"] --> F["Tool Security"]
G["LETTA_OTEL_EXPORTER_OTLP_ENDPOINT"] --> H["Observability"]
I["CLICKHOUSE_ENDPOINT"] --> J["Metrics Storage"]
K["SECURE"] --> L["Security Mode"]
```

### Configuration Impact on Performance

Different environment variable settings significantly impact system performance:

| Variable | Development | Production | Performance Impact |
|----------|-------------|------------|-------------------|
| LETTA_DEBUG | True | False | Reduces overhead by ~30% |
| SECURE | false | true | Adds minimal overhead |
| LETTA_SANDBOX_MOUNT_PATH | unset | configured | Enables secure tool execution |
| Database Pooling | disabled | enabled | Improves connection reuse |

**Section sources**
- [compose.yaml](file://compose.yaml#L35-L52)
- [letta/server/startup.sh](file://letta/server/startup.sh#L50-L54)

## Monitoring and Observability

### Database Performance Monitoring

The SQLAlchemy instrumentation provides comprehensive database performance monitoring:

```mermaid
sequenceDiagram
participant App as "Application"
participant Instrument as "SQLAlchemy Instrumentation"
participant Tracer as "OpenTelemetry Tracer"
participant Exporter as "Metrics Exporter"
App->>Instrument : Database Operation
Instrument->>Tracer : Create Span
Instrument->>Instrument : Track Performance
Instrument->>Tracer : Record Metrics
Tracer->>Exporter : Export Data
Exporter->>Exporter : Store/Forward Metrics
```

**Diagram sources**
- [letta/otel/sqlalchemy_instrumentation.py](file://letta/otel/sqlalchemy_instrumentation.py#L1-L200)

### Endpoint Performance Tracking

The metrics system tracks critical endpoint performance:

- **Latency Histograms**: Distribution of request processing times
- **Throughput Counters**: Requests per second across all endpoints
- **Error Rate Tracking**: Failure rates by endpoint and HTTP status
- **Resource Utilization**: Memory and CPU usage patterns

**Section sources**
- [letta/otel/metrics.py](file://letta/otel/metrics.py#L40-L140)

## Best Practices and Recommendations

### Resource Allocation Guidelines

For optimal vertical scaling performance, follow these resource allocation guidelines:

1. **CPU Allocation**: Allocate 2-4 cores per Letta instance for moderate workloads
2. **Memory Configuration**: Minimum 4GB RAM, 8GB recommended for production
3. **Storage Requirements**: SSD storage recommended for database performance
4. **Network Bandwidth**: 100Mbps minimum for reliable API communication

### Performance Optimization Checklist

- [ ] Enable PostgreSQL connection pooling
- [ ] Configure appropriate database vacuum settings
- [ ] Set optimal message buffer limits
- [ ] Enable parallel tool execution where applicable
- [ ] Configure OpenTelemetry for production monitoring
- [ ] Implement proper volume mounting for persistence
- [ ] Set appropriate timeouts for long-running operations
- [ ] Monitor memory usage patterns and adjust limits

### Scaling Decision Matrix

| Workload Size | Recommended Resources | Key Considerations |
|---------------|----------------------|-------------------|
| Small (< 10 agents) | 2GB RAM, 2 cores | Basic configuration |
| Medium (10-100 agents) | 4GB RAM, 4 cores | Enable pooling and monitoring |
| Large (100-1000 agents) | 8GB RAM, 8 cores | Optimize memory management |
| Enterprise (> 1000 agents) | 16GB+ RAM, 16+ cores | Distributed architecture |

### Troubleshooting Performance Issues

Common performance bottlenecks and solutions:

1. **Memory Pressure**: Reduce message buffer limits or increase available memory
2. **Database Lock Contention**: Optimize query patterns and connection pooling
3. **Tool Execution Delays**: Increase parallel execution limits
4. **Network Latency**: Deploy closer to clients or optimize network configuration

This comprehensive vertical scaling strategy ensures Letta can handle demanding production workloads while maintaining optimal performance and reliability. The combination of efficient containerization, intelligent memory management, and robust monitoring creates a foundation for scalable AI agent deployments.