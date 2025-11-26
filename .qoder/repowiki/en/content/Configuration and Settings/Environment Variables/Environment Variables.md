# Environment Variables

<cite>
**Referenced Files in This Document**
- [settings.py](file://letta/settings.py)
- [environment_variables.py](file://letta/schemas/environment_variables.py)
- [crypto_utils.py](file://letta/helpers/crypto_utils.py)
- [secret.py](file://letta/schemas/secret.py)
- [compose.yaml](file://compose.yaml)
- [dev-compose.yaml](file://dev-compose.yaml)
- [sandbox_config.py](file://letta/schemas/sandbox_config.py)
- [otel-collector-config-clickhouse-prod.yaml](file://otel/otel-collector-config-clickhouse-prod.yaml)
- [app.py](file://letta/server/rest_api/app.py)
- [log.py](file://letta/log.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Environment Variable Naming Conventions](#environment-variable-naming-conventions)
3. [Configuration Classes Overview](#configuration-classes-overview)
4. [Database Connection Parameters](#database-connection-parameters)
5. [LLM Provider Keys](#llm-provider-keys)
6. [Tool Sandbox Credentials](#tool-sandbox-credentials)
7. [Telemetry Settings](#telemetry-settings)
8. [Security and Encryption](#security-and-encryption)
9. [Deployment Scenarios](#deployment-scenarios)
10. [Fallback Mechanisms and Default Values](#fallback-mechanisms-and-default-values)
11. [Best Practices](#best-practices)
12. [Troubleshooting](#troubleshooting)

## Introduction

Letta's configuration system uses environment variables to manage all aspects of application behavior, from database connections to LLM provider credentials and telemetry settings. The system is built around Pydantic's SettingsConfigDict, which provides automatic environment variable mapping with flexible naming conventions and type safety.

The configuration system consists of five main classes that handle different aspects of the application:
- **Settings**: Core application configuration
- **ModelSettings**: LLM provider and model configuration
- **ToolSettings**: Tool sandbox and external service credentials
- **TelemetrySettings**: Observability and monitoring configuration
- **LogSettings**: Logging and debugging configuration

## Environment Variable Naming Conventions

Letta follows a hierarchical naming convention using the `letta_` prefix for most configuration variables, with specialized prefixes for different subsystems:

```mermaid
graph TD
A["Environment Variable Prefixes"] --> B["letta_ (Main Settings)"]
A --> C["letta_model_ (Model Settings)"]
A --> D["letta_tool_ (Tool Settings)"]
A --> E["letta_telemetry_ (Telemetry Settings)"]
A --> F["letta_logging_ (Log Settings)"]
A --> G["letta_summarizer_ (Summarizer Settings)"]
B --> B1["Database: pg_*"]
B --> B2["Security: encryption_key"]
B --> B3["Debugging: debug"]
C --> C1["API Keys: *_api_key"]
C --> C2["Base URLs: *_base_url"]
C --> C3["Provider Settings: *_api_*"]
D --> D1["Sandbox: e2b_api_key, modal_*"]
D --> D2["Search: tavily_api_key, exa_api_key"]
D --> D3["Timeouts: tool_*_timeout"]
```

**Diagram sources**
- [settings.py](file://letta/settings.py#L230-L416)

### Pydantic SettingsConfigDict Behavior

The configuration system uses Pydantic's SettingsConfigDict to handle environment variable mapping with several key features:

- **Automatic Case Conversion**: Environment variables are automatically converted from uppercase to lowercase snake_case
- **Prefix Matching**: Variables are matched based on their prefix (e.g., `LETTA_PG_URI` becomes `pg_uri`)
- **Type Validation**: Automatic type conversion and validation
- **Alias Support**: Support for multiple environment variable names for the same setting

**Section sources**
- [settings.py](file://letta/settings.py#L60-L416)

## Configuration Classes Overview

### Settings Class

The main Settings class handles core application configuration with the `letta_` prefix:

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| `letta_dir` | Path | Main Letta directory | `~/.letta` |
| `debug` | bool | Enable debug mode | `False` |
| `environment` | str | Application environment | `None` |
| `pg_uri` | str | PostgreSQL connection URI | Auto-generated |
| `pg_db` | str | Database name | `None` |
| `pg_user` | str | Database user | `None` |
| `pg_password` | str | Database password | `None` |
| `pg_host` | str | Database host | `None` |
| `pg_port` | int | Database port | `None` |

### ModelSettings Class

Handles LLM provider configurations with provider-specific prefixes:

| Provider | API Key Variable | Base URL Variable | Additional Settings |
|----------|------------------|-------------------|-------------------|
| OpenAI | `openai_api_key` | `openai_api_base` | Custom base URL support |
| Anthropic | `anthropic_api_key` | - | Sonnet 1M context window |
| Google Gemini | `gemini_api_key` | `gemini_base_url` | Force minimum thinking budget |
| Azure OpenAI | `azure_api_key` | `azure_base_url` | API version support |
| Groq | `groq_api_key` | - | High-performance inference |
| DeepSeek | `deepseek_api_key` | - | Specialized models |
| Ollama | `ollama_base_url` | - | Local model serving |
| Together AI | `together_api_key` | - | Community models |

### ToolSettings Class

Manages tool sandbox and external service configurations:

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| `e2b_api_key` | str | E2B sandbox API key | `None` |
| `modal_token_id` | str | Modal sandbox token ID | `None` |
| `modal_token_secret` | str | Modal sandbox token secret | `None` |
| `tavily_api_key` | str | Tavily search API key | `None` |
| `exa_api_key` | str | Exa search API key | `None` |
| `tool_exec_dir` | str | Tool execution directory | `None` |
| `tool_sandbox_timeout` | float | Sandbox timeout (seconds) | `180.0` |

### TelemetrySettings Class

Controls observability and monitoring integration:

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| `enable_datadog` | bool | Enable Datadog profiling | `False` |
| `otel_exporter_otlp_endpoint` | str | OTLP exporter endpoint | `None` |
| `datadog_agent_host` | str | Datadog agent hostname | `"localhost"` |
| `datadog_agent_port` | int | Datadog agent port | `8126` |
| `datadog_service_name` | str | Service name for Datadog | `"letta-server"` |

**Section sources**
- [settings.py](file://letta/settings.py#L17-L459)

## Database Connection Parameters

Letta supports multiple ways to configure database connections, with flexible fallback mechanisms:

### Connection Methods

```mermaid
flowchart TD
A["Database Connection"] --> B{"pg_uri provided?"}
B --> |Yes| C["Use pg_uri directly"]
B --> |No| D{"Individual params provided?"}
D --> |Yes| E["Build URI from params"]
D --> |No| F["Use default localhost"]
E --> G["postgresql+pg8000://user:password@host:port/db"]
F --> H["postgresql+pg8000://letta:letta@localhost:5432/letta"]
```

**Diagram sources**
- [settings.py](file://letta/settings.py#L362-L379)

### Database Configuration Variables

| Variable | Type | Description | Example |
|----------|------|-------------|---------|
| `pg_uri` | str | Complete connection URI | `postgresql://user:pass@host:5432/db` |
| `pg_db` | str | Database name | `letta` |
| `pg_user` | str | Database username | `letta` |
| `pg_password` | str | Database password | `letta` |
| `pg_host` | str | Database host | `localhost` |
| `pg_port` | int | Database port | `5432` |

### Connection Pool Configuration

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| `pg_pool_size` | int | Initial pool size | `25` |
| `pg_max_overflow` | int | Maximum overflow connections | `10` |
| `pg_pool_timeout` | int | Connection timeout (seconds) | `30` |
| `pg_pool_recycle` | int | Connection recycling time (seconds) | `1800` |
| `pg_echo` | bool | Enable SQL logging | `False` |
| `pool_pre_ping` | bool | Pre-ping connections | `True` |

**Section sources**
- [settings.py](file://letta/settings.py#L249-L264)

## LLM Provider Keys

Letta supports multiple LLM providers through environment variables. Each provider has specific configuration requirements:

### Provider Configuration Matrix

```mermaid
graph LR
subgraph "Cloud Providers"
A[OpenAI<br/>openai_api_key]
B[Anthropic<br/>anthropic_api_key]
C[Gemini<br/>gemini_api_key]
D[Azure OpenAI<br/>azure_api_key]
end
subgraph "Local/Community"
E[Groq<br/>groq_api_key]
F[DeepSeek<br/>deepseek_api_key]
G[Together AI<br/>together_api_key]
H[Ollama<br/>ollama_base_url]
end
subgraph "Enterprise"
I[VLLM<br/>vllm_api_base]
J[OpenLLM<br/>openllm_api_key]
end
```

**Diagram sources**
- [settings.py](file://letta/settings.py#L113-L200)

### Authentication Patterns

Most providers follow a consistent pattern for API key configuration:

```python
# Example pattern for API key retrieval
api_key = model_settings.provider_api_key or os.environ.get(f"{provider.upper()}_API_KEY")
```

### Special Provider Features

#### Anthropic Sonnet 1M Context Window
- Variable: `ANTHROPIC_SONNET_1M`
- Enables 1M token context window for Claude Sonnet 4/4.5
- Adds beta context header and adjusts context window

#### Azure OpenAI Configuration
- Requires separate API key, base URL, and API version
- Default API version: `2024-09-01-preview`
- Supports multiple deployments under the same subscription

#### Google Gemini Advanced Features
- `gemini_force_minimum_thinking_budget`: Forces minimum thinking budget
- `gemini_max_retries`: Configurable retry logic (default: 5)

**Section sources**
- [settings.py](file://letta/settings.py#L113-L200)

## Tool Sandbox Credentials

Letta supports multiple sandbox environments for secure tool execution:

### Sandbox Types and Configuration

```mermaid
classDiagram
class SandboxType {
<<enumeration>>
E2B
MODAL
LOCAL
}
class ToolSettings {
+e2b_api_key : str
+e2b_sandbox_template_id : str
+modal_token_id : str
+modal_token_secret : str
+tool_exec_dir : str
+tool_sandbox_timeout : float
+sandbox_type() SandboxType
+modal_sandbox_enabled() bool
}
class SandboxConfig {
+type : SandboxType
+config : Dict
+get_e2b_config() E2BSandboxConfig
+get_modal_config() ModalSandboxConfig
+get_local_config() LocalSandboxConfig
}
ToolSettings --> SandboxType
SandboxConfig --> SandboxType
```

**Diagram sources**
- [settings.py](file://letta/settings.py#L17-L58)
- [sandbox_config.py](file://letta/schemas/sandbox_config.py#L88-L112)

### Sandbox Credential Variables

| Sandbox Type | Variables | Description |
|--------------|-----------|-------------|
| E2B | `e2b_api_key` | API key for E2B sandbox |
| E2B | `e2b_sandbox_template_id` | Template ID for sandbox environment |
| Modal | `modal_token_id` | Token ID for Modal sandbox |
| Modal | `modal_token_secret` | Token secret for Modal sandbox |
| Local | `tool_exec_dir` | Directory for local sandbox execution |
| Local | `tool_sandbox_timeout` | Timeout for local sandbox operations |

### Sandbox Selection Logic

The system automatically selects the appropriate sandbox type based on available credentials:

```mermaid
flowchart TD
A["Sandbox Selection"] --> B{"E2B API Key Available?"}
B --> |Yes| C["Use E2B Sandbox"]
B --> |No| D{"Modal Credentials Available?"}
D --> |Yes| E["Use Modal Sandbox"]
D --> |No| F["Use Local Sandbox"]
```

**Diagram sources**
- [settings.py](file://letta/settings.py#L47-L58)

**Section sources**
- [settings.py](file://letta/settings.py#L17-L58)
- [sandbox_config.py](file://letta/schemas/sandbox_config.py#L88-L112)

## Telemetry Settings

Letta integrates with multiple observability platforms for monitoring and debugging:

### Telemetry Architecture

```mermaid
graph TB
subgraph "Telemetry Sources"
A[Application Metrics]
B[Tracing Data]
C[Logging Events]
D[Performance Metrics]
end
subgraph "Exporters"
E[OTLP Exporter]
F[Datadog Exporter]
G[JSON File Exporter]
end
subgraph "Monitoring Platforms"
H[ClickHouse]
I[Datadog]
J[Custom Dashboards]
end
A --> E
B --> E
C --> E
D --> E
E --> H
E --> I
E --> J
F --> I
G --> J
```

**Diagram sources**
- [settings.py](file://letta/settings.py#L276-L288)
- [otel-collector-config-clickhouse-prod.yaml](file://otel/otel-collector-config-clickhouse-prod.yaml#L56-L90)

### OpenTelemetry Configuration

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| `otel_exporter_otlp_endpoint` | str | OTLP collector endpoint | `None` |
| `otel_preferred_temporality` | int | Metric temporality preference | `1` (Delta) |
| `disable_tracing` | bool | Disable OpenTelemetry tracing | `False` |

### Datadog Integration

| Variable | Type | Description | Default |
|----------|------|-------------|---------|
| `enable_datadog` | bool | Enable Datadog profiling | `False` |
| `datadog_agent_host` | str | Datadog agent hostname | `"localhost"` |
| `datadog_agent_port` | int | Datadog agent port | `8126` |
| `datadog_service_name` | str | Service name for Datadog | `"letta-server"` |
| `datadog_profiling_enabled` | bool | Enable profiling | `False` |
| `datadog_profiling_memory_enabled` | bool | Enable memory profiling | `False` |
| `datadog_profiling_heap_enabled` | bool | Enable heap profiling | `False` |

### Source Code Integration

For advanced debugging and code navigation:

| Variable | Type | Description |
|----------|------|-------------|
| `datadog_git_repository_url` | str | Git repository URL |
| `datadog_git_commit_sha` | str | Git commit SHA |
| `datadog_main_package` | str | Primary Python package |

**Section sources**
- [settings.py](file://letta/settings.py#L276-L288)
- [settings.py](file://letta/settings.py#L416-L450)
- [app.py](file://letta/server/rest_api/app.py#L229-L253)

## Security and Encryption

Letta implements robust encryption for sensitive data using AES-256-GCM encryption:

### Encryption Architecture

```mermaid
sequenceDiagram
participant App as Application
participant Crypto as CryptoUtils
participant Secret as Secret Class
participant DB as Database
App->>Secret : Secret.from_plaintext(value)
Secret->>Crypto : encrypt(plaintext, master_key)
Crypto->>Crypto : derive_key(master_key, salt)
Crypto->>Crypto : generate_iv()
Crypto->>Crypto : AES-256-GCM encryption
Crypto-->>Secret : encrypted_data
Secret-->>App : Secret object
App->>DB : Store encrypted value
DB-->>App : Confirmation
App->>DB : Retrieve encrypted value
DB-->>App : encrypted_data
App->>Secret : Secret.from_encrypted(data)
Secret->>Crypto : decrypt(encrypted_data, master_key)
Crypto-->>Secret : plaintext
Secret-->>App : Decrypted value
```

**Diagram sources**
- [crypto_utils.py](file://letta/helpers/crypto_utils.py#L32-L148)
- [secret.py](file://letta/schemas/secret.py#L41-L192)

### Encryption Configuration

| Variable | Type | Description | Security Level |
|----------|------|-------------|----------------|
| `encryption_key` | str | Master encryption key | High |

### Key Features

- **AES-256-GCM**: Industry-standard encryption with authentication
- **PBKDF2 Key Derivation**: Secure key generation from master key
- **Random Salts**: Unique salts for each encryption operation
- **Automatic Fallback**: Stores as plaintext if no encryption key
- **Migration Support**: Handles both encrypted and plaintext data

### Security Best Practices

1. **Strong Master Key**: Use a cryptographically strong random key
2. **Environment Variables**: Store encryption key in environment variables
3. **Key Rotation**: Plan for periodic key rotation
4. **Backup Strategy**: Ensure encryption key backups for disaster recovery
5. **Access Control**: Restrict access to encryption keys

**Section sources**
- [crypto_utils.py](file://letta/helpers/crypto_utils.py#L13-L148)
- [secret.py](file://letta/schemas/secret.py#L41-L192)
- [settings.py](file://letta/settings.py#L338-L340)

## Deployment Scenarios

### Development Environment

Development deployments typically use simplified configurations:

```yaml
# Development .env file
LETTA_DEBUG=true
LETTA_PG_URI=postgresql://letta:letta@localhost:5432/letta_dev
OPENAI_API_KEY=your-development-key
ANTHROPIC_API_KEY=your-anthropic-key
```

### Production Environment

Production deployments require enhanced security and monitoring:

```yaml
# Production .env file
LETTA_DEBUG=false
LETTA_PG_URI=${LETTA_PG_URI}
LETTA_ENCRYPTION_KEY=${LETTA_ENCRYPTION_KEY}

# LLM Providers
OPENAI_API_KEY=${OPENAI_API_KEY}
ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
GEMINI_API_KEY=${GEMINI_API_KEY}

# Telemetry
LETTA_OTEL_EXPORTER_OTLP_ENDPOINT=https://otel-collector:4317
DATADOG_API_KEY=${DATADOG_API_KEY}
ENABLE_DATADOG=true

# Database
LETTA_PG_DB=${LETTA_PG_DB}
LETTA_PG_USER=${LETTA_PG_USER}
LETTA_PG_PASSWORD=${LETTA_PG_PASSWORD}
LETTA_PG_HOST=${LETTA_PG_HOST}
LETTA_PG_PORT=${LETTA_PG_PORT}
```

### Container Deployment

Docker Compose configurations demonstrate environment variable usage:

```yaml
# compose.yaml environment variables
environment:
  - LETTA_PG_URI=${LETTA_PG_URI:-postgresql://letta:letta@letta-db:5432/letta}
  - LETTA_DEBUG=True
  - OPENAI_API_KEY=${OPENAI_API_KEY}
  - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
  - LETTA_OTEL_EXPORTER_OTLP_ENDPOINT=${LETTA_OTEL_EXPORTER_OTLP_ENDPOINT}
```

### Multi-Environment Setup

```mermaid
graph TB
subgraph "Development"
A[Dev .env<br/>Debug enabled<br/>Local DB]
end
subgraph "Staging"
B[Staging .env<br/>Test data<br/>Limited monitoring]
end
subgraph "Production"
C[Prod .env<br/>Full security<br/>Complete monitoring]
end
A --> D[Shared Configuration]
B --> D
C --> D
D --> E[Environment Variables]
D --> F[Encryption Keys]
D --> G[Provider Credentials]
```

**Section sources**
- [compose.yaml](file://compose.yaml#L32-L49)
- [dev-compose.yaml](file://dev-compose.yaml#L30-L49)

## Fallback Mechanisms and Default Values

Letta implements sophisticated fallback mechanisms to ensure reliable operation:

### Configuration Priority Order

```mermaid
flowchart TD
A["Configuration Loading"] --> B["Environment Variables"]
B --> C["Settings Class Properties"]
C --> D["Default Values"]
B --> E["Provider-specific fallbacks"]
E --> F["API key from model_settings"]
F --> G["API key from os.environ"]
D --> H["Database connection"]
H --> I["Built-in defaults"]
I --> J["localhost:5432/letta"]
```

**Diagram sources**
- [settings.py](file://letta/settings.py#L361-L379)

### Database Connection Fallbacks

The system provides multiple fallback mechanisms for database connections:

1. **Direct URI**: Highest priority (`pg_uri`)
2. **Parameter Combination**: Build URI from individual parameters
3. **Default Connection**: Fallback to localhost with default credentials

### LLM Provider Fallbacks

Each LLM provider follows a consistent fallback pattern:

```python
# Example fallback logic
api_key = model_settings.provider_api_key or os.environ.get(f"{provider.upper()}_API_KEY")
```

### Default Value Matrix

| Setting Category | Primary Source | Fallback Source | Default Value |
|------------------|----------------|-----------------|---------------|
| Database URI | `pg_uri` | Individual params | Built-in URI |
| API Keys | Provider settings | Environment | `None` |
| Debug Mode | Settings | Environment | `False` |
| Encryption | Settings | Environment | `None` |
| Sandbox Type | Credentials | Manual selection | `LOCAL` |

**Section sources**
- [settings.py](file://letta/settings.py#L361-L379)

## Best Practices

### Security Best Practices

1. **Environment Variables Only**: Never hardcode sensitive information
2. **Encryption Key Management**: Use strong, randomly generated encryption keys
3. **Access Control**: Restrict file permissions on configuration files
4. **Rotation Schedule**: Implement regular credential rotation
5. **Audit Logging**: Monitor access to sensitive configurations

### Configuration Management

1. **Version Control**: Track configuration changes in version control
2. **Environment Separation**: Use separate configurations for different environments
3. **Validation**: Validate configuration values at startup
4. **Documentation**: Document all environment variables and their purposes
5. **Testing**: Test configuration changes in staging environments

### Performance Optimization

1. **Connection Pooling**: Tune database connection pool settings
2. **Timeout Configuration**: Set appropriate timeouts for external services
3. **Caching**: Implement caching for frequently accessed configurations
4. **Monitoring**: Monitor configuration-related performance metrics

### Troubleshooting Guidelines

1. **Debug Mode**: Enable debug mode for development troubleshooting
2. **Logging**: Use structured logging for better visibility
3. **Health Checks**: Implement health checks for critical configurations
4. **Graceful Degradation**: Design for graceful degradation when configurations fail

## Troubleshooting

### Common Issues and Solutions

#### Database Connection Problems

**Issue**: Cannot connect to PostgreSQL database
**Solution**: 
- Verify `pg_uri` or individual connection parameters
- Check database server availability
- Validate credentials and network connectivity

#### LLM Provider Authentication

**Issue**: API key authentication failures
**Solution**:
- Verify API key format and validity
- Check provider-specific requirements
- Review rate limits and quotas

#### Encryption Key Issues

**Issue**: Encryption/decryption failures
**Solution**:
- Verify `encryption_key` is set correctly
- Check key format and length
- Ensure consistent key across deployments

#### Sandbox Configuration

**Issue**: Tool execution sandbox failures
**Solution**:
- Verify sandbox credentials
- Check sandbox type selection logic
- Review timeout and resource limits

### Diagnostic Commands

```bash
# Check environment variables
printenv | grep LETTA_

# Verify database connectivity
psql "$LETTA_PG_URI" -c "SELECT 1"

# Test API connectivity
curl -H "Authorization: Bearer $OPENAI_API_KEY" \
     https://api.openai.com/v1/models
```

### Monitoring and Alerting

Implement monitoring for:
- Configuration loading success
- Database connection health
- LLM provider availability
- Encryption/decryption performance
- Sandbox execution success rates

**Section sources**
- [settings.py](file://letta/settings.py#L230-L459)