# Server Settings

<cite>
**Referenced Files in This Document**
- [settings.py](file://letta/settings.py)
- [app.py](file://letta/server/rest_api/app.py)
- [constants.py](file://letta/server/constants.py)
- [db.py](file://letta/server/db.py)
- [redis_client.py](file://letta/data_sources/redis_client.py)
- [scheduler.py](file://letta/jobs/scheduler.py)
- [metrics.py](file://letta/otel/metrics.py)
- [db_pool_monitoring.py](file://letta/otel/db_pool_monitoring.py)
- [log.py](file://letta/log.py)
- [otel-collector-config-clickhouse-prod.yaml](file://otel/otel-collector-config-clickhouse-prod.yaml)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Settings Architecture](#settings-architecture)
3. [FastAPI Server Configuration](#fastapi-server-configuration)
4. [Database Connection Pooling](#database-connection-pooling)
5. [Redis Configuration](#redis-configuration)
6. [Telemetry and Observability](#telemetry-and-observability)
7. [Event Loop and Background Processing](#event-loop-and-background-processing)
8. [CORS and Client Communication](#cors-and-client-communication)
9. [Performance Tuning Guidelines](#performance-tuning-guidelines)
10. [Production Deployment Best Practices](#production-deployment-best-practices)

## Introduction

Letta's server settings provide comprehensive configuration options for controlling the FastAPI server behavior, database connections, caching systems, telemetry, and background processing. The Settings class serves as the central configuration hub, allowing fine-grained control over all server operational parameters.

The configuration system is built around Pydantic's BaseSettings, providing type safety, environment variable support, and validation. All settings can be configured through environment variables prefixed with `LETTA_` or through configuration files.

## Settings Architecture

The Settings class is the primary configuration container for Letta's server settings, inheriting from Pydantic's BaseSettings to provide robust validation and environment variable integration.

```mermaid
classDiagram
class Settings {
+Path letta_dir
+bool debug
+list cors_origins
+str environment
+bool enable_keepalive
+float keepalive_interval
+bool enable_cancellation_aware_streaming
+str default_llm_handle
+str default_embedding_handle
+str pg_db
+str pg_user
+str pg_password
+str pg_host
+int pg_port
+str pg_uri
+int pg_pool_size
+int pg_max_overflow
+int pg_pool_timeout
+int pg_pool_recycle
+bool pg_echo
+bool pool_pre_ping
+bool pool_use_lifo
+bool disable_sqlalchemy_pooling
+int db_max_concurrent_sessions
+str redis_host
+int redis_port
+int multi_agent_send_message_max_retries
+int multi_agent_send_message_timeout
+int multi_agent_concurrent_sends
+str otel_exporter_otlp_endpoint
+int otel_preferred_temporality
+bool disable_tracing
+bool llm_api_logging
+bool track_last_agent_run
+bool track_errored_messages
+bool track_stop_reason
+bool track_agent_run
+bool track_provider_trace
+int uvicorn_workers
+bool uvicorn_reload
+int uvicorn_timeout_keep_alive
+bool use_uvloop
+bool use_granian
+bool sqlalchemy_tracing
+int event_loop_threadpool_max_workers
+bool enable_db_pool_monitoring
+int db_pool_monitoring_interval
+bool enable_batch_job_polling
+int poll_running_llm_batches_interval_seconds
+int poll_lock_retry_interval_seconds
+int batch_job_polling_lookback_weeks
+int batch_job_polling_batch_size
+float llm_request_timeout_seconds
+float llm_stream_timeout_seconds
+bool enable_pinecone
+str pinecone_api_key
+str pinecone_source_index
+str pinecone_agent_index
+bool upsert_pinecone_indices
+bool use_tpuf
+str tpuf_api_key
+str tpuf_region
+bool embed_all_messages
+str encryption_key
+int file_processing_timeout_minutes
+str file_processing_timeout_error_message
+str default_base_url
+str default_token
+bool use_letta_v1_agent
+int archival_memory_token_limit
+bool no_default_actor
+letta_pg_uri() str
+letta_pg_uri_no_default() str
+database_engine() DatabaseChoice
+plugin_register_dict() dict
}
class TelemetrySettings {
+bool enable_datadog
+str datadog_agent_host
+int datadog_agent_port
+str datadog_service_name
+bool datadog_profiling_enabled
+bool datadog_profiling_memory_enabled
+bool datadog_profiling_heap_enabled
+str datadog_git_repository_url
+str datadog_git_commit_sha
+str datadog_main_package
}
class LogSettings {
+bool debug
+bool json_logging
+str log_level
+Path letta_log_path
+bool verbose_telemetry_logging
}
class TestSettings {
+Path letta_dir
}
Settings --> TelemetrySettings : "contains"
Settings --> LogSettings : "contains"
Settings --> TestSettings : "contains"
```

**Diagram sources**
- [settings.py](file://letta/settings.py#L230-L460)

**Section sources**
- [settings.py](file://letta/settings.py#L230-L460)

## FastAPI Server Configuration

Letta's FastAPI server configuration controls the underlying ASGI server behavior, including worker processes, hot reloading, and connection timeouts.

### Uvicorn Worker Configuration

The `uvicorn_workers` setting controls the number of worker processes for handling concurrent requests:

| Setting | Default Value | Description | Production Recommendation |
|---------|---------------|-------------|---------------------------|
| `uvicorn_workers` | 1 | Number of Uvicorn worker processes | 2-4 for production, based on CPU cores |
| `uvicorn_reload` | False | Enable hot reloading during development | False for production |
| `uvicorn_timeout_keep_alive` | 5 | Keep-alive timeout in seconds | 5-30 for production |

### Event Loop Optimization

The server supports uvloop integration for improved asyncio performance:

| Setting | Default Value | Description | Performance Impact |
|---------|---------------|-------------|-------------------|
| `use_uvloop` | False | Enable uvloop as asyncio event loop | 20-40% performance improvement |
| `use_granian` | False | Use Granian for workers | Higher performance, experimental |

### Server Startup and Lifecycle

The server initialization includes several critical components:

```mermaid
sequenceDiagram
participant App as "FastAPI App"
participant Middleware as "Middleware Stack"
participant DB as "Database Pool"
participant Scheduler as "Job Scheduler"
participant Watchdog as "Event Loop Watchdog"
App->>Middleware : Initialize CORS & Logging
App->>DB : Setup Connection Pool
App->>Scheduler : Start Background Jobs
App->>Watchdog : Monitor Event Loop
App->>App : Mount Routes & Middleware
Note over App,Watchdog : Server Ready for Requests
```

**Diagram sources**
- [app.py](file://letta/server/rest_api/app.py#L132-L200)
- [scheduler.py](file://letta/jobs/scheduler.py#L79-L208)

**Section sources**
- [settings.py](file://letta/settings.py#L289-L296)
- [app.py](file://letta/server/rest_api/app.py#L655-L749)

## Database Connection Pooling

Letta implements sophisticated database connection pooling using SQLAlchemy's AsyncAdaptedQueuePool with configurable parameters for optimal performance and reliability.

### Pool Configuration Parameters

| Parameter | Default Value | Description | Tuning Guidelines |
|-----------|---------------|-------------|-------------------|
| `pg_pool_size` | 25 | Initial pool size | 25-50 for moderate load |
| `pg_max_overflow` | 10 | Maximum overflow connections | 10-20% of pool_size |
| `pg_pool_timeout` | 30 | Connection timeout in seconds | 30-60 seconds |
| `pg_pool_recycle` | 1800 | Recycle connections after seconds | 1800-3600 seconds |
| `pool_pre_ping` | True | Verify connections before use | True for reliability |
| `pool_use_lifo` | True | Use LIFO for connection allocation | True for efficiency |

### Connection Pool Monitoring

Letta provides comprehensive connection pool monitoring with real-time metrics:

```mermaid
flowchart TD
A[Database Pool] --> B[Connection Events]
B --> C[Connect]
B --> D[Checkout]
B --> E[Checkin]
B --> F[Invalidate]
C --> G[Metrics Collection]
D --> G
E --> G
F --> G
G --> H[Pool Statistics]
G --> I[Performance Metrics]
H --> J[Active Connections]
H --> K[Available Connections]
H --> L[Overflow Count]
I --> M[Connection Events Counter]
I --> N[Pool Gauge Metrics]
```

**Diagram sources**
- [db_pool_monitoring.py](file://letta/otel/db_pool_monitoring.py#L35-L309)

### Pool Configuration Implementation

The database configuration dynamically adjusts based on the `disable_sqlalchemy_pooling` setting:

```mermaid
flowchart TD
A[Database Configuration] --> B{disable_sqlalchemy_pooling?}
B --> |True| C[NullPool - No Pooling]
B --> |False| D[AsyncAdaptedQueuePool]
D --> E[Configure Pool Size]
D --> F[Configure Max Overflow]
D --> G[Configure Timeout]
D --> H[Configure Recycle]
E --> I[Connection Pool Ready]
F --> I
G --> I
H --> I
C --> J[Direct Connections]
```

**Diagram sources**
- [db.py](file://letta/server/db.py#L19-L46)

**Section sources**
- [settings.py](file://letta/settings.py#L256-L264)
- [db.py](file://letta/server/db.py#L19-L46)
- [db_pool_monitoring.py](file://letta/otel/db_pool_monitoring.py#L35-L309)

## Redis Configuration

Letta integrates Redis for caching, message brokering, and distributed coordination. The Redis client provides connection pooling and automatic reconnection capabilities.

### Redis Connection Settings

| Setting | Default Value | Description | Production Considerations |
|---------|---------------|-------------|---------------------------|
| `redis_host` | None | Redis server hostname | Use private network or VPN |
| `redis_port` | 6379 | Redis server port | Standard port or custom |
| `max_connections` | 50 | Maximum pooled connections | Based on concurrent clients |
| `socket_timeout` | 5 | Socket timeout in seconds | 5-10 seconds typical |
| `socket_connect_timeout` | 5 | Connection timeout in seconds | 5 seconds typical |
| `retry_on_timeout` | True | Retry operations on timeout | True for reliability |

### Redis Client Architecture

```mermaid
classDiagram
class AsyncRedisClient {
+ConnectionPool pool
+Redis _client
+Lock _lock
+get_client() Redis
+set(key, value) bool
+get(key) Any
+exists(*keys) int
+sadd(key, *members) int
+smismember(key, values) list
+delete(*keys) int
+check_inclusion_and_exclusion(member, group) bool
+create_inclusion_exclusion_keys(group) void
+scard(key) int
+smembers(key) Set
+srem(key, *members) int
+xadd(stream, fields, id, maxlen, approximate) str
+xread(streams, count, block) List
+xrange(stream, start, end, count) List
+xrevrange(stream, start, end, count) List
+xlen(stream) int
+xdel(stream, *ids) int
+xinfo_stream(stream) Dict
+xtrim(stream, maxlen, approximate) int
}
class NoopAsyncRedisClient {
+set(key, value) bool
+get(key) Any
+exists(*keys) int
+sadd(key, *members) int
+smismember(key, values) list
+delete(*keys) int
+check_inclusion_and_exclusion(member, group) bool
+create_inclusion_exclusion_keys(group) void
+scard(key) int
+smembers(key) Set
+srem(key, *members) int
+xadd(stream, fields, id, maxlen, approximate) str
+xread(streams, count, block) List
+xrange(stream, start, end, count) List
+xrevrange(stream, start, end, count) List
+xlen(stream) int
+xdel(stream, *ids) int
+xinfo_stream(stream) Dict
+xtrim(stream, maxlen, approximate) int
}
AsyncRedisClient --|> NoopAsyncRedisClient : "fallback"
```

**Diagram sources**
- [redis_client.py](file://letta/data_sources/redis_client.py#L41-L457)

### Redis Stream Operations

Letta uses Redis streams for message brokering and event distribution:

```mermaid
sequenceDiagram
participant Producer as "Message Producer"
participant Stream as "Redis Stream"
participant Consumer as "Message Consumer"
participant Handler as "Message Handler"
Producer->>Stream : XADD message
Stream->>Consumer : XREAD new messages
Consumer->>Handler : Process message
Handler->>Consumer : Acknowledge completion
Consumer->>Stream : XDEL processed messages
Note over Producer,Handler : Message lifecycle with acknowledgment
```

**Diagram sources**
- [redis_client.py](file://letta/data_sources/redis_client.py#L414-L436)

**Section sources**
- [settings.py](file://letta/settings.py#L266-L267)
- [redis_client.py](file://letta/data_sources/redis_client.py#L41-L457)

## Telemetry and Observability

Letta provides comprehensive telemetry and observability through OpenTelemetry integration, with optional Datadog APM and profiling capabilities.

### OpenTelemetry Configuration

| Setting | Default Value | Description | Monitoring Scope |
|---------|---------------|-------------|------------------|
| `otel_exporter_otlp_endpoint` | None | OTLP endpoint URL | Distributed tracing |
| `otel_preferred_temporality` | 1 | Metric export temporality | Delta/Cumulative |
| `disable_tracing` | False | Disable OTEL tracing | Performance impact |

### Datadog Integration

Datadog provides APM, profiling, and source code integration:

| Setting | Default Value | Description | Production Use |
|---------|---------------|-------------|----------------|
| `enable_datadog` | False | Enable Datadog profiling | True for production |
| `datadog_agent_host` | localhost | Datadog agent hostname | Private network |
| `datadog_agent_port` | 8126 | Datadog trace agent port | 8126 typical |
| `datadog_service_name` | letta-server | Service name for profiling | Environment-specific |
| `datadog_profiling_enabled` | False | Enable profiling | True for performance |
| `datadog_profiling_memory_enabled` | False | Memory profiling | True for memory issues |
| `datadog_profiling_heap_enabled` | False | Heap profiling | True for memory leaks |

### Telemetry Architecture

```mermaid
flowchart TD
A[Letta Server] --> B[OpenTelemetry SDK]
A --> C[Datadog Agent]
B --> D[OTLP Exporter]
D --> E[OTLP Endpoint]
C --> F[Traces]
C --> G[Metrics]
C --> H[Profiling]
F --> I[Datadog Backend]
G --> I
H --> I
I --> J[Dashboards]
I --> K[Alerts]
I --> L[Source Code Integration]
```

**Diagram sources**
- [log.py](file://letta/log.py#L61-L247)
- [otel-collector-config-clickhouse-prod.yaml](file://otel/otel-collector-config-clickhouse-prod.yaml#L56-L90)

### Metrics and Monitoring

Letta collects various metrics for performance monitoring:

```mermaid
classDiagram
class MetricRegistry {
+Counter db_pool_connection_events_counter
+Gauge db_pool_connections_checked_out_gauge
+Gauge db_pool_connections_available_gauge
+Gauge db_pool_connections_overflow_gauge
+Gauge db_pool_connections_total_gauge
+Histogram message_cost
+Histogram message_output_tokens
+Histogram endpoint_e2e_ms_histogram
}
class DatabasePoolMonitor {
+dict _active_connections
+setup_monitoring(engine, engine_name) void
+_setup_pool_listeners(pool, engine_name) void
+_get_pool_stats(pool) dict
}
MetricRegistry --> DatabasePoolMonitor : "monitors"
```

**Diagram sources**
- [metric_registry.py](file://letta/otel/metric_registry.py#L199-L236)
- [db_pool_monitoring.py](file://letta/otel/db_pool_monitoring.py#L35-L309)

**Section sources**
- [settings.py](file://letta/settings.py#L277-L288)
- [settings.py](file://letta/settings.py#L418-L449)
- [metrics.py](file://letta/otel/metrics.py#L1-L75)
- [log.py](file://letta/log.py#L61-L247)

## Event Loop and Background Processing

Letta implements sophisticated background processing capabilities with cron job scheduling and thread pool management for handling asynchronous operations.

### Cron Job Configuration

| Setting | Default Value | Description | Use Case |
|---------|---------------|-------------|----------|
| `enable_batch_job_polling` | False | Enable batch job processing | Large-scale operations |
| `poll_running_llm_batches_interval_seconds` | 300 | Polling interval for LLM batches | Real-time processing |
| `poll_lock_retry_interval_seconds` | 480 | Lock retry interval | Distributed coordination |
| `batch_job_polling_lookback_weeks` | 2 | Lookback period for jobs | Historical processing |
| `batch_job_polling_batch_size` | None | Batch processing size | Memory optimization |

### Thread Pool Management

| Setting | Default Value | Description | Performance Impact |
|---------|---------------|-------------|-------------------|
| `event_loop_threadpool_max_workers` | 43 | Maximum thread pool workers | Concurrency control |

### Background Processing Architecture

```mermaid
flowchart TD
A[Scheduler Manager] --> B[Leader Election]
B --> C{Is Leader?}
C --> |Yes| D[Start Scheduler]
C --> |No| E[Wait for Leader]
D --> F[Cron Jobs]
D --> G[Batch Processing]
D --> H[Lock Management]
F --> I[Periodic Tasks]
G --> J[LLM Batch Jobs]
H --> K[Advisory Locks]
I --> L[Task Execution]
J --> L
K --> L
L --> M[Job Completion]
M --> N[Metrics Reporting]
```

**Diagram sources**
- [scheduler.py](file://letta/jobs/scheduler.py#L79-L208)

### Lock Management for Distributed Systems

Letta uses PostgreSQL advisory locks for distributed coordination:

```mermaid
sequenceDiagram
participant App1 as "App Instance 1"
participant App2 as "App Instance 2"
participant DB as "PostgreSQL"
participant Scheduler as "Scheduler"
App1->>DB : Try Acquire Lock
App2->>DB : Try Acquire Lock
DB-->>App1 : Lock Acquired
DB-->>App2 : Lock Failed
App1->>Scheduler : Start as Leader
App2->>App2 : Wait for Lock
Note over App1,Scheduler : App1 becomes leader
Note over App2 : App2 waits for lock release
App1->>DB : Release Lock
App2->>DB : Acquire Lock
App2->>Scheduler : Start as Leader
```

**Diagram sources**
- [scheduler.py](file://letta/jobs/scheduler.py#L79-L208)

**Section sources**
- [settings.py](file://letta/settings.py#L311-L317)
- [scheduler.py](file://letta/jobs/scheduler.py#L79-L208)

## CORS and Client Communication

Letta implements comprehensive CORS (Cross-Origin Resource Sharing) configuration to enable secure client-server communication across different domains.

### CORS Configuration

The CORS middleware is configured with flexible origin handling:

| Setting | Default Values | Description | Security Considerations |
|---------|----------------|-------------|------------------------|
| `cors_origins` | ["http://letta.localhost", "http://localhost:8283", "http://localhost:8083", "http://localhost:3000", "http://localhost:4200"] | Allowed origins | Restrict to known domains |
| `allow_credentials` | True | Allow credentials | Enable for authenticated requests |
| `allow_methods` | ["*"] | Allowed HTTP methods | Limit to necessary methods |
| `allow_headers` | ["*"] | Allowed headers | Restrict to necessary headers |

### WebSocket Configuration

| Setting | Default Value | Description | Connection Behavior |
|---------|---------------|-------------|-------------------|
| `WS_DEFAULT_PORT` | 8282 | WebSocket server port | Separate from REST API |
| `WS_CLIENT_TIMEOUT` | 30 | Client timeout in seconds | Prevent hanging connections |

### Client Communication Patterns

```mermaid
sequenceDiagram
participant Client as "Web Client"
participant CORS as "CORS Middleware"
participant Auth as "Authentication"
participant API as "REST API"
participant WS as "WebSocket Server"
Client->>CORS : Preflight Request
CORS-->>Client : Allow Origin/Headers
Client->>Auth : Authentication Headers
Auth->>API : Authorized Request
API-->>Client : Response
Client->>WS : WebSocket Connection
WS-->>Client : Connection Established
Client->>WS : Messages
WS-->>Client : Real-time Updates
```

**Diagram sources**
- [constants.py](file://letta/server/constants.py#L1-L6)
- [app.py](file://letta/server/rest_api/app.py#L572-L578)

### Keep-Alive Configuration

Letta supports configurable keep-alive settings for maintaining persistent connections:

| Setting | Default Value | Description | Production Use |
|---------|---------------|-------------|----------------|
| `enable_keepalive` | True | Enable SSE keepalive | True for streaming |
| `keepalive_interval` | 50.0 | Keepalive interval in seconds | 30-60 seconds |

**Section sources**
- [settings.py](file://letta/settings.py#L196-L209)
- [constants.py](file://letta/server/constants.py#L1-L6)
- [app.py](file://letta/server/rest_api/app.py#L572-L578)

## Performance Tuning Guidelines

Optimizing Letta's server performance requires careful consideration of multiple factors including database configuration, connection pooling, and resource allocation.

### Database Performance Tuning

#### Connection Pool Optimization

```mermaid
flowchart TD
A[Load Assessment] --> B[Calculate Peak Connections]
B --> C[Configure Pool Size]
C --> D[Set Overflow Limit]
D --> E[Adjust Timeout Values]
E --> F[Test Under Load]
F --> G{Performance OK?}
G --> |No| H[Tune Parameters]
G --> |Yes| I[Deploy Configuration]
H --> C
```

#### Recommended Pool Settings by Load

| Load Level | Pool Size | Max Overflow | Timeout | Recycle |
|------------|-----------|--------------|---------|---------|
| Low (< 100 concurrent) | 10-20 | 5-10 | 30s | 1800s |
| Medium (100-500 concurrent) | 25-50 | 10-20 | 30s | 1800s |
| High (> 500 concurrent) | 50-100 | 20-50 | 60s | 3600s |

### Memory and CPU Optimization

#### Event Loop Configuration

- **uvloop**: Enable for significant performance improvements
- **Granian**: Consider for higher throughput scenarios
- **Thread Pool**: Adjust based on CPU core count

#### Redis Optimization

- **Connection Pool**: Size based on concurrent client count
- **Timeouts**: Balance between responsiveness and stability
- **Persistence**: Configure based on data importance

### Network and I/O Optimization

#### Keep-Alive Settings

```mermaid
flowchart LR
A[Client Request] --> B[Keep-Alive Check]
B --> C{Timeout Reached?}
C --> |No| D[Continue Connection]
C --> |Yes| E[Send Keep-Alive]
E --> F[Reset Timer]
F --> D
D --> G[Process Request]
G --> H[Response Sent]
```

#### WebSocket Optimization

- **Timeout Values**: Adjust based on expected connection duration
- **Compression**: Enable for large message payloads
- **Heartbeat**: Configure based on network reliability

**Section sources**
- [settings.py](file://letta/settings.py#L256-L264)
- [settings.py](file://letta/settings.py#L289-L296)

## Production Deployment Best Practices

Deploying Letta in production requires careful attention to security, monitoring, and scalability considerations.

### Security Configuration

#### Environment Variables

```bash
# Database Configuration
LETTA_PG_URI=postgresql://user:pass@host:5432/db
LETTA_DISABLE_SQLALCHEMY_POOLING=false

# Redis Configuration  
LETTA_REDIS_HOST=redis.internal
LETTA_REDIS_PORT=6379

# Telemetry Configuration
LETTA_TELEMETRY_OTEL_EXPORTER_OTLP_ENDPOINT=https://otel-collector:4317
LETTA_TELEMETRY_ENABLE_DATADOG=true
LETTA_TELEMETRY_DATADOG_AGENT_HOST=datadog-agent
LETTA_TELEMETRY_DATADOG_SERVICE_NAME=letta-production

# Security Settings
LETTA_DEBUG=false
LETTA_CORS_ORIGINS=https://your-domain.com,https://app.your-domain.com
```

#### Network Security

- **Firewall Rules**: Restrict database and Redis access to internal networks
- **SSL/TLS**: Enable HTTPS for all external communications
- **Authentication**: Use strong authentication mechanisms
- **CORS Origins**: Restrict to known domains only

### Monitoring and Alerting

#### Essential Metrics

```mermaid
mindmap
root((Monitoring))
Performance
Response Times
Throughput
Error Rates
Resources
CPU Usage
Memory Usage
Database Connections
Redis Connections
Business
Active Agents
Message Volume
API Usage
Infrastructure
Health Checks
Disk Space
Network Latency
```

#### Alert Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Database Connections | >80% | >95% | Scale pool or investigate |
| Response Time | >2s | >5s | Investigate bottlenecks |
| Error Rate | >1% | >5% | Review logs and metrics |
| Memory Usage | >80% | >90% | Scale or optimize |

### Scaling Strategies

#### Horizontal Scaling

```mermaid
flowchart TD
A[Load Balancer] --> B[Server Instance 1]
A --> C[Server Instance 2]
A --> D[Server Instance N]
B --> E[Shared Database]
C --> E
D --> E
B --> F[Redis Cluster]
C --> F
D --> F
G[Message Queue] --> B
G --> C
G --> D
```

#### Vertical Scaling

- **CPU**: Increase cores for concurrent processing
- **Memory**: Scale for larger agent states and caches
- **Storage**: Use SSD storage for database performance
- **Network**: Ensure adequate bandwidth for streaming

### Backup and Recovery

#### Database Backup Strategy

- **Frequency**: Daily full backups, hourly incremental
- **Retention**: 30 days full, 7 days incremental
- **Verification**: Regular restore testing
- **Encryption**: Encrypt backups in transit and at rest

#### Redis Persistence

- **RDB Snapshots**: Configure based on data importance
- **AOF**: Enable for durability guarantees
- **Replication**: Set up master-slave configuration

### Maintenance Procedures

#### Rolling Updates

1. **Pre-deployment**: Validate configuration changes
2. **Deployment**: Update instances one by one
3. **Validation**: Monitor metrics and logs
4. **Rollback**: Prepare rollback procedures

#### Capacity Planning

- **Growth Trends**: Monitor usage patterns
- **Resource Limits**: Set capacity thresholds
- **Scaling Triggers**: Automate scaling decisions
- **Cost Optimization**: Review resource utilization

**Section sources**
- [settings.py](file://letta/settings.py#L230-L460)
- [app.py](file://letta/server/rest_api/app.py#L572-L578)