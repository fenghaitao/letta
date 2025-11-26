# Technology Stack & Dependencies

<cite>
**Referenced Files in This Document**
- [pyproject.toml](file://pyproject.toml)
- [Dockerfile](file://Dockerfile)
- [compose.yaml](file://compose.yaml)
- [dev-compose.yaml](file://dev-compose.yaml)
- [docker-compose-vllm.yaml](file://docker-compose-vllm.yaml)
- [alembic/env.py](file://alembic/env.py)
- [letta/orm/sqlalchemy_base.py](file://letta/orm/sqlalchemy_base.py)
- [letta/server/db.py](file://letta/server/db.py)
- [letta/llm_api/openai_client.py](file://letta/llm_api/openai_client.py)
- [letta/llm_api/anthropic_client.py](file://letta/llm_api/anthropic_client.py)
- [letta/local_llm/ollama/api.py](file://letta/local_llm/ollama/api.py)
- [letta/settings.py](file://letta/settings.py)
- [letta/database_utils.py](file://letta/database_utils.py)
- [letta/data_sources/redis_client.py](file://letta/data_sources/redis_client.py)
- [letta/server/rest_api/app.py](file://letta/server/rest_api/app.py)
- [letta/server/rest_api/redis_stream_manager.py](file://letta/server/rest_api/redis_stream_manager.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Core Technology Foundation](#core-technology-foundation)
3. [Database Architecture](#database-architecture)
4. [LLM Integration Layer](#llm-integration-layer)
5. [Containerization Strategy](#containerization-strategy)
6. [Local LLM Infrastructure](#local-llm-infrastructure)
7. [Streaming & Real-time Communication](#streaming--real-time-communication)
8. [Dependency Management](#dependency-management)
9. [Version Compatibility Matrix](#version-compatibility-matrix)
10. [Extensibility Framework](#extensibility-framework)
11. [Infrastructure Requirements](#infrastructure-requirements)
12. [Best Practices](#best-practices)

## Introduction

The Letta platform is built on a modern, scalable technology stack designed to support advanced AI agent functionality with robust database persistence, flexible LLM integration, and efficient real-time communication. The architecture emphasizes modularity, extensibility, and performance while maintaining compatibility across diverse deployment scenarios.

## Core Technology Foundation

### FastAPI Framework

The platform leverages FastAPI as its primary web framework, providing:

- **High-performance ASGI server**: Built on uvicorn with optional Granian support for production deployments
- **Automatic API documentation**: Swagger UI and ReDoc generation for seamless API exploration
- **Type-safe request/response handling**: Pydantic validation ensures data integrity
- **Async-first architecture**: Full asynchronous support for concurrent operations

```mermaid
graph TB
subgraph "Web Layer"
FastAPI[FastAPI Application]
Middleware[Custom Middleware]
CORS[CORS Middleware]
end
subgraph "Routing Layer"
Routers[API Routers]
Auth[Authentication]
Validation[Request Validation]
end
subgraph "Service Layer"
Managers[Service Managers]
Validators[Schema Validators]
end
FastAPI --> Middleware
Middleware --> CORS
CORS --> Routers
Routers --> Auth
Auth --> Validation
Validation --> Managers
Managers --> Validators
```

**Diagram sources**
- [letta/server/rest_api/app.py](file://letta/server/rest_api/app.py#L1-L711)

### SQLAlchemy ORM

The platform uses SQLAlchemy with async support for comprehensive database operations:

- **Async ORM**: Full async/await support with SQLAlchemy 2.0
- **Type safety**: Pydantic integration for schema validation
- **Connection pooling**: Optimized connection management with configurable pools
- **Database abstraction**: Support for PostgreSQL, SQLite, and other dialects

```mermaid
classDiagram
class SqlalchemyBase {
+String id
+DateTime created_at
+DateTime updated_at
+Boolean is_deleted
+list_async() SqlalchemyBase[]
+read_async() SqlalchemyBase
+create_async() SqlalchemyBase
+update_async() SqlalchemyBase
+delete_async() SqlalchemyBase
}
class DatabaseRegistry {
+async_session() AsyncSession
+get_db_async() AsyncGenerator
+close_db() None
}
class AsyncSession {
+commit() None
+rollback() None
+close() None
+flush() None
}
SqlalchemyBase --> DatabaseRegistry : uses
DatabaseRegistry --> AsyncSession : manages
```

**Diagram sources**
- [letta/orm/sqlalchemy_base.py](file://letta/orm/sqlalchemy_base.py#L63-L816)
- [letta/server/db.py](file://letta/server/db.py#L51-L106)

### Pydantic Data Validation

Pydantic serves as the primary validation framework:

- **Schema definition**: Type-safe data models with automatic validation
- **Serialization**: Seamless conversion between Python objects and JSON
- **Error handling**: Comprehensive error reporting with detailed feedback
- **Nested validation**: Support for complex nested data structures

### Alembic Database Migrations

Alembic provides robust database migration capabilities:

- **Version control**: Track database schema changes systematically
- **Cross-database support**: Compatible with PostgreSQL and SQLite
- **Automated migrations**: Schema updates without downtime
- **Rollback capability**: Safe rollback mechanisms for production environments

**Section sources**
- [alembic/env.py](file://alembic/env.py#L1-L94)
- [letta/orm/sqlalchemy_base.py](file://letta/orm/sqlalchemy_base.py#L1-L816)

## Database Architecture

### Primary Database Options

The platform supports multiple database backends with PostgreSQL as the recommended production option:

#### PostgreSQL with pgvector
- **Vector embeddings**: Native support for similarity search
- **Advanced indexing**: GIN and GiST indexes for performance
- **Connection pooling**: asyncpg driver with optimized pools
- **Scalability**: Horizontal scaling support with replication

#### SQLite for Development
- **Zero-configuration**: Single-file database
- **Development convenience**: Quick setup for testing
- **Embedded deployment**: Suitable for edge deployments

#### Redis for Streaming
- **Real-time messaging**: High-performance pub/sub capabilities
- **Stream processing**: Ordered message queues with TTL
- **Memory efficiency**: Optimized for high-throughput scenarios

```mermaid
graph TB
subgraph "Database Layer"
PG[(PostgreSQL<br/>Production)]
SQLite[(SQLite<br/>Development)]
Redis[(Redis<br/>Streaming)]
end
subgraph "ORM Layer"
SQLAlchemy[SQLAlchemy ORM]
Pydantic[Pydantic Schemas]
end
subgraph "Connection Management"
Pool[Connection Pool]
Migration[Alembic Migrations]
end
SQLAlchemy --> PG
SQLAlchemy --> SQLite
SQLAlchemy --> Redis
Pydantic --> SQLAlchemy
Pool --> PG
Migration --> PG
```

**Diagram sources**
- [letta/server/db.py](file://letta/server/db.py#L1-L106)
- [letta/database_utils.py](file://letta/database_utils.py#L1-L162)

### Database Configuration

The platform provides flexible database configuration through environment variables and settings:

| Configuration Parameter | Description | Default Value |
|------------------------|-------------|---------------|
| `LETТА_PG_URI` | PostgreSQL connection string | `postgresql://letta:letta@localhost:5432/letta` |
| `LETТА_PG_DB` | Database name | `letta` |
| `LETТА_PG_USER` | Database user | `letta` |
| `LETТА_PG_PASSWORD` | Database password | `letta` |
| `LETТА_PG_HOST` | Database host | `localhost` |
| `LETТА_PG_PORT` | Database port | `5432` |
| `PG_POOL_SIZE` | Connection pool size | `25` |
| `PG_MAX_OVERFLOW` | Maximum overflow connections | `10` |

**Section sources**
- [letta/settings.py](file://letta/settings.py#L250-L264)
- [letta/database_utils.py](file://letta/database_utils.py#L145-L162)

## LLM Integration Layer

### Supported LLM Providers

The platform integrates with multiple LLM providers through a unified interface:

#### OpenAI-Compatible Providers
- **OpenAI**: GPT-3.5, GPT-4, GPT-4o, o1 series
- **OpenRouter**: Access to diverse model ecosystem
- **Azure OpenAI**: Enterprise-grade OpenAI deployment
- **Groq**: High-speed inference for OpenAI-compatible models

#### Native Provider Integrations
- **Anthropic**: Claude 3.5 Sonnet, Claude 4, and reasoning models
- **Google AI**: Gemini Pro and Vertex AI
- **AWS Bedrock**: Multi-provider LLM access
- **Together AI**: Community-driven model hosting

#### Local LLM Support
- **Ollama**: Local model deployment with automatic model management
- **vLLM**: High-performance inference with GPU acceleration
- **KoboldCpp**: CPU-based local inference
- **LM Studio**: Windows-native local inference

```mermaid
graph TB
subgraph "LLM Interface Layer"
LLMClient[LLM Client Base]
OpenAIClient[OpenAI Client]
AnthropicClient[Anthropic Client]
LocalLLM[Local LLM Proxy]
end
subgraph "Provider Layer"
OpenAI[OpenAI API]
Anthropic[Anthropic API]
Azure[Azure OpenAI]
Ollama[Ollama Local]
VLLM[vLLM Inference]
end
subgraph "Configuration"
LLMConfig[LLM Configuration]
ModelSettings[Model Settings]
end
LLMClient --> OpenAIClient
LLMClient --> AnthropicClient
LLMClient --> LocalLLM
OpenAIClient --> OpenAI
OpenAIClient --> Azure
AnthropicClient --> Anthropic
LocalLLM --> Ollama
LocalLLM --> VLLM
LLMConfig --> ModelSettings
```

**Diagram sources**
- [letta/llm_api/openai_client.py](file://letta/llm_api/openai_client.py#L1-L1083)
- [letta/llm_api/anthropic_client.py](file://letta/llm_api/anthropic_client.py#L1-L1279)

### Advanced LLM Features

The platform supports sophisticated LLM capabilities:

#### Structured Output
- **JSON schema validation**: Automatic response formatting
- **Type safety**: Pydantic model integration
- **Error recovery**: Fallback mechanisms for malformed responses

#### Reasoning Models
- **Extended thinking**: Claude's interleaved reasoning
- **Step-by-step reasoning**: GPT-4o and o1 series support
- **Effort control**: Fine-grained reasoning budget management

#### Tool Calling
- **Parallel tool execution**: Concurrent tool invocation
- **Function calling**: Standardized function interface
- **Tool composition**: Complex multi-step workflows

**Section sources**
- [letta/llm_api/openai_client.py](file://letta/llm_api/openai_client.py#L1-L800)
- [letta/llm_api/anthropic_client.py](file://letta/llm_api/anthropic_client.py#L1-L800)

## Containerization Strategy

### Docker Architecture

The platform uses a multi-stage Docker build for optimal performance and security:

#### Builder Stage
- **Base image**: pgvector v0.5.1 for PostgreSQL with vector extensions
- **Python installation**: Optimized Python environment
- **Package installation**: Dependency resolution with uv for speed

#### Runtime Stage
- **Minimal footprint**: Alpine-based runtime image
- **Security**: Non-root user execution
- **Optimization**: Compiled binaries and optimized dependencies

```mermaid
graph TB
subgraph "Build Process"
Builder[Docker Builder]
Dependencies[Install Dependencies]
App[Copy Application]
end
subgraph "Runtime Image"
Runtime[Alpine Runtime]
NodeJS[Node.js 22]
OpenTelemetry[OTEL Collector]
Services[Application Services]
end
Builder --> Dependencies
Dependencies --> App
App --> Runtime
Runtime --> NodeJS
Runtime --> OpenTelemetry
Runtime --> Services
```

**Diagram sources**
- [Dockerfile](file://Dockerfile#L1-L89)

### Docker Compose Configurations

The platform provides multiple Docker Compose configurations for different deployment scenarios:

#### Production Configuration
- **Separate services**: Database, server, and Nginx reverse proxy
- **Environment isolation**: Separate network namespaces
- **Persistent storage**: Volume mounts for data persistence
- **Load balancing**: Nginx for traffic distribution

#### Development Configuration
- **Integrated setup**: Single container with hot reloading
- **Development tools**: Debugging and profiling support
- **Volume mounting**: Live code updates
- **Simplified networking**: Direct service communication

#### vLLM Inference Configuration
- **GPU acceleration**: NVIDIA runtime support
- **Model caching**: Persistent model storage
- **High-performance serving**: Optimized vLLM deployment
- **Resource management**: GPU resource reservation

**Section sources**
- [compose.yaml](file://compose.yaml#L1-L66)
- [dev-compose.yaml](file://dev-compose.yaml#L1-L49)
- [docker-compose-vllm.yaml](file://docker-compose-vllm.yaml#L1-L36)

## Local LLM Infrastructure

### Ollama Integration

Ollama provides seamless local LLM deployment:

#### Automatic Model Management
- **Model discovery**: Automatic detection of installed models
- **Context window detection**: Dynamic context length calculation
- **Resource allocation**: Memory and CPU optimization
- **Model switching**: Hot-swappable model support

#### Configuration Options
- **Endpoint configuration**: Flexible endpoint setup
- **Authentication**: API key management
- **Model selection**: Dynamic model choice
- **Performance tuning**: Optimization parameters

```mermaid
sequenceDiagram
participant Client as Client
participant Ollama as Ollama API
participant LocalLLM as Local LLM Handler
participant Model as Ollama Model
Client->>LocalLLM : Request Completion
LocalLLM->>Ollama : List Available Models
Ollama-->>LocalLLM : Model List
LocalLLM->>Ollama : Select Model
Ollama->>Model : Load Model
Model-->>Ollama : Model Ready
LocalLLM->>Ollama : Generate Completion
Ollama->>Model : Process Prompt
Model-->>Ollama : Generate Response
Ollama-->>LocalLLM : Completion Result
LocalLLM-->>Client : Formatted Response
```

**Diagram sources**
- [letta/local_llm/ollama/api.py](file://letta/local_llm/ollama/api.py#L1-L89)

### vLLM Integration

vLLM provides high-performance local inference:

#### GPU Acceleration
- **CUDA support**: NVIDIA GPU acceleration
- **Model parallelism**: Multi-GPU model distribution
- **Quantization**: INT8/INT4 model support
- **Memory optimization**: Efficient memory usage

#### Deployment Features
- **Auto-tool calling**: Automatic function invocation
- **Streaming support**: Real-time response generation
- **Batch processing**: Concurrent request handling
- **Model serving**: REST API endpoint

**Section sources**
- [docker-compose-vllm.yaml](file://docker-compose-vllm.yaml#L16-L35)
- [letta/local_llm/ollama/api.py](file://letta/local_llm/ollama/api.py#L11-L89)

## Streaming & Real-time Communication

### Redis Streaming Architecture

The platform uses Redis for high-performance real-time communication:

#### Stream-Based Architecture
- **Message queuing**: Ordered message delivery
- **TTL management**: Automatic cleanup of old messages
- **Batch processing**: Efficient bulk operations
- **Consumer groups**: Parallel message processing

#### SSE (Server-Sent Events) Implementation
- **Real-time updates**: Browser-compatible streaming
- **Connection management**: Automatic reconnection handling
- **Error recovery**: Graceful failure handling
- **Backpressure control**: Flow control mechanisms

```mermaid
graph TB
subgraph "Streaming Layer"
RedisStream[Redis Streams]
SSEWriter[SSE Stream Writer]
SSEGenerator[SSE Stream Generator]
end
subgraph "Client Layer"
Browser[Web Browser]
WebSocket[WebSocket Client]
REST[REST API]
end
subgraph "Processing Layer"
RunManager[Run Manager]
MessageManager[Message Manager]
EventLoop[Event Loop]
end
RunManager --> RedisStream
MessageManager --> SSEWriter
SSEWriter --> SSEGenerator
SSEGenerator --> Browser
SSEGenerator --> WebSocket
SSEGenerator --> REST
EventLoop --> RunManager
```

**Diagram sources**
- [letta/server/rest_api/redis_stream_manager.py](file://letta/server/rest_api/redis_stream_manager.py#L1-L445)
- [letta/data_sources/redis_client.py](file://letta/data_sources/redis_client.py#L41-L457)

### Performance Optimizations

#### Connection Pooling
- **Redis connections**: Optimized connection reuse
- **Database pools**: SQLAlchemy connection management
- **HTTP clients**: Reusable client instances
- **Resource limits**: Configurable limits per service

#### Batching Strategies
- **Write batching**: Redis pipeline operations
- **Read batching**: Bulk message retrieval
- **Compression**: Efficient data serialization
- **Caching**: Response caching strategies

**Section sources**
- [letta/server/rest_api/redis_stream_manager.py](file://letta/server/rest_api/redis_stream_manager.py#L23-L70)
- [letta/data_sources/redis_client.py](file://letta/data_sources/redis_client.py#L41-L457)

## Dependency Management

### Poetry Package Management

The platform uses Poetry for comprehensive dependency management:

#### Core Dependencies
- **FastAPI**: Web framework with automatic documentation
- **SQLAlchemy**: ORM with async support
- **Pydantic**: Data validation and serialization
- **Alembic**: Database migrations
- **OpenAI**: LLM API integration
- **Anthropic**: Claude API integration

#### Optional Dependencies
- **PostgreSQL**: Production database support
- **Redis**: Streaming and caching
- **vLLM**: High-performance inference
- **Ollama**: Local model deployment
- **Testing**: Comprehensive test suite

```mermaid
graph TB
subgraph "Core Dependencies"
FastAPI[FastAPI >=0.115.6]
SQLAlchemy[SQLAlchemy >=2.0.41]
Pydantic[Pydantic >=2.10.6]
Alembic[Alembic >=1.13.3]
end
subgraph "LLM Dependencies"
OpenAI[OpenAI >=1.99.9]
Anthropic[Anthropic >=0.75.0]
GoogleAI[Google GenAI >=1.15.0]
Mistral[Mistral AI >=1.8.1]
end
subgraph "Infrastructure"
Redis[Redis >=6.2.0]
PostgreSQL[PostgreSQL Drivers]
VLLM[vLLM Support]
Ollama[Ollama Integration]
end
subgraph "Development"
Testing[Test Dependencies]
Linting[Code Quality]
Formatting[Code Formatting]
end
FastAPI --> SQLAlchemy
SQLAlchemy --> Pydantic
OpenAI --> Pydantic
Anthropic --> Pydantic
Redis --> FastAPI
PostgreSQL --> SQLAlchemy
```

**Diagram sources**
- [pyproject.toml](file://pyproject.toml#L11-L75)

### Version Pinning Strategy

The platform employs strategic version pinning:

#### Major Version Stability
- **Python**: 3.11+ requirement for async features
- **FastAPI**: Stable 0.115.x series
- **SQLAlchemy**: 2.0+ for async support
- **Pydantic**: 2.10+ for performance improvements

#### Minor Version Flexibility
- **LLM SDKs**: Latest compatible versions
- **Infrastructure**: Latest stable releases
- **Development**: Latest features and bug fixes

**Section sources**
- [pyproject.toml](file://pyproject.toml#L10-L11)

## Version Compatibility Matrix

### Python Compatibility

| Python Version | Status | Notes |
|---------------|--------|-------|
| 3.11 | ✅ Supported | Minimum requirement |
| 3.12 | ✅ Supported | Recommended |
| 3.13 | ⚠️ Experimental | Testing compatibility |
| 3.14 | ❌ Not Supported | Future compatibility |

### Database Compatibility

| Database | Version | Status | Notes |
|----------|---------|--------|-------|
| PostgreSQL | 14+ | ✅ Production | Recommended 16+ |
| PostgreSQL | 13 | ⚠️ Legacy | Limited support |
| SQLite | 3.35+ | ✅ Development | Full feature support |
| Redis | 6.0+ | ✅ Streaming | Required for production |

### LLM Provider Compatibility

| Provider | Status | Models Supported | Features |
|----------|--------|------------------|----------|
| OpenAI | ✅ Production | GPT-3.5, GPT-4, GPT-4o, o1 | All features |
| Anthropic | ✅ Production | Claude 3.5 Sonnet, Claude 4 | Reasoning support |
| Azure OpenAI | ✅ Production | All OpenAI models | Enterprise features |
| Google AI | ✅ Production | Gemini Pro | Vision support |
| Ollama | ✅ Development | Local models | Model switching |
| vLLM | ✅ Development | Custom models | GPU acceleration |

## Extensibility Framework

### Adding New LLM Providers

The platform provides a standardized interface for adding new LLM providers:

#### Provider Interface
- **Base class inheritance**: Extend LLMClientBase
- **Standardized methods**: Request, stream, and batch processing
- **Error handling**: Consistent exception patterns
- **Configuration**: Environment-based setup

#### Integration Steps
1. **Create provider class**: Implement required methods
2. **Add configuration**: Environment variables and defaults
3. **Test integration**: Unit and integration tests
4. **Documentation**: Provider-specific documentation

### Database Backend Extensions

The platform supports custom database backends:

#### ORM Integration
- **SQLAlchemy dialects**: Custom database drivers
- **Connection pooling**: Optimized connection management
- **Migration support**: Alembic schema management
- **Performance tuning**: Database-specific optimizations

### Plugin Architecture

The platform supports extensible plugins:

#### Plugin Types
- **Tool plugins**: Custom tool integration
- **Provider plugins**: New LLM provider support
- **Storage plugins**: Alternative storage backends
- **Middleware plugins**: Request/response processing

**Section sources**
- [letta/settings.py](file://letta/settings.py#L250-L264)

## Infrastructure Requirements

### Hardware Requirements

#### Minimum Requirements
- **CPU**: 2 cores (4+ recommended)
- **Memory**: 4GB RAM (8GB+ recommended)
- **Storage**: 10GB free space
- **Network**: Internet connectivity for LLM APIs

#### Production Requirements
- **CPU**: 8+ cores for high throughput
- **Memory**: 16GB+ for large models
- **Storage**: SSD with 100GB+ free space
- **GPU**: NVIDIA GPU for vLLM inference

#### Local LLM Requirements
- **Ollama**: 2GB+ RAM per model
- **vLLM**: 8GB+ VRAM for medium models
- **GPU**: NVIDIA RTX 3060+ for optimal performance

### Software Prerequisites

#### Operating Systems
- **Linux**: Ubuntu 20.04+, CentOS 7+
- **macOS**: macOS 10.15+
- **Windows**: Windows 10+ with WSL2

#### Container Runtime
- **Docker**: 20.10+
- **Docker Compose**: 2.0+
- **Podman**: Alternative container runtime

#### Additional Tools
- **Node.js**: 22+ for OpenTelemetry
- **Git**: For source code management
- **Python**: 3.11+ for development

## Best Practices

### Performance Optimization

#### Database Optimization
- **Connection pooling**: Configure appropriate pool sizes
- **Indexing**: Create indexes for frequently queried columns
- **Query optimization**: Use SQLAlchemy's query builder efficiently
- **Monitoring**: Enable database performance monitoring

#### LLM Optimization
- **Model selection**: Choose appropriate models for use cases
- **Context management**: Optimize context window usage
- **Caching**: Implement response caching strategies
- **Batch processing**: Group requests for better throughput

#### Streaming Optimization
- **Buffer sizing**: Optimize Redis stream buffer sizes
- **TTL management**: Configure appropriate expiration times
- **Compression**: Use efficient serialization formats
- **Monitoring**: Track streaming performance metrics

### Security Considerations

#### API Security
- **Authentication**: Implement proper authentication mechanisms
- **Rate limiting**: Prevent abuse and ensure fair usage
- **Input validation**: Validate all user inputs
- **Error handling**: Secure error message presentation

#### Data Protection
- **Encryption**: Encrypt sensitive data at rest and in transit
- **Access control**: Implement role-based access control
- **Audit logging**: Maintain comprehensive audit trails
- **Backup strategies**: Regular backup and recovery procedures

### Monitoring and Observability

#### Metrics Collection
- **Performance metrics**: Response times and throughput
- **Resource utilization**: CPU, memory, and disk usage
- **Error rates**: Failure rates and error patterns
- **Business metrics**: User engagement and feature usage

#### Distributed Tracing
- **Request tracing**: Track requests across microservices
- **Span correlation**: Link related operations
- **Performance analysis**: Identify bottlenecks and optimizations
- **Alerting**: Set up alerts for critical issues

**Section sources**
- [letta/settings.py](file://letta/settings.py#L276-L288)