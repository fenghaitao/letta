# Configuration and Settings

<cite>
**Referenced Files in This Document**   
- [config.py](file://letta/config.py)
- [settings.py](file://letta/settings.py)
- [llm_config.py](file://letta/schemas/llm_config.py)
- [embedding_config.py](file://letta/schemas/embedding_config.py)
- [environment_variables.py](file://letta/schemas/environment_variables.py)
- [secret.py](file://letta/schemas/secret.py)
- [crypto_utils.py](file://letta/helpers/crypto_utils.py)
- [server.py](file://letta/server/server.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Environment Variables and Service Endpoints](#environment-variables-and-service-endpoints)
3. [Settings Hierarchy and Override Mechanisms](#settings-hierarchy-and-override-mechanisms)
4. [LLM Configuration Options](#llm-configuration-options)
5. [Embedding Configuration](#embedding-configuration)
6. [Agent-Specific Settings](#agent-specific-settings)
7. [Memory Configuration](#memory-configuration)
8. [Tool Selection and Execution Parameters](#tool-selection-and-execution-parameters)
9. [Security Considerations for Sensitive Credentials](#security-considerations-for-sensitive-credentials)
10. [Environment Separation](#environment-separation)
11. [Performance Tuning](#performance-tuning)
12. [Configuration Change Monitoring](#configuration-change-monitoring)

## Introduction
Letta's configuration system provides a comprehensive framework for managing application settings, model parameters, and security credentials. The system is designed to support multiple deployment environments while maintaining security and flexibility. Configuration is managed through a combination of environment variables, configuration files, and database-stored settings, with a clear hierarchy that determines which values take precedence. This document details all configurable parameters, including database connections, API keys, service endpoints, LLM settings, agent configurations, and security practices for managing sensitive information.

**Section sources**
- [config.py](file://letta/config.py#L40-L311)
- [settings.py](file://letta/settings.py#L230-L460)

## Environment Variables and Service Endpoints
Letta utilizes environment variables as the primary mechanism for configuring service endpoints and API credentials. The system supports integration with multiple LLM providers, embedding services, and external tools through dedicated environment variables. These variables are organized by service type and provider, allowing for flexible configuration of the AI stack.

The main categories of environment variables include:

**LLM Provider Credentials:**
- `OPENAI_API_KEY`: Authentication token for OpenAI services
- `ANTHROPIC_API_KEY`: API key for Anthropic's Claude models
- `GEMINI_API_KEY`: Credentials for Google's Gemini models
- `TOGETHER_API_KEY`: API key for Together AI services
- `GROQ_API_KEY`: Authentication for Groq inference services
- `DEEPSEEK_API_KEY`: Credentials for DeepSeek models
- `XAI_API_KEY`: API key for xAI (Grok) services

**Database Configuration:**
- `LETTA_PG_URI`: Full PostgreSQL connection URI
- `PG_DB`, `PG_USER`, `PG_PASSWORD`, `PG_HOST`, `PG_PORT`: Individual PostgreSQL connection parameters
- `REDIS_HOST`, `REDIS_PORT`: Redis server connection details

**External Service Integrations:**
- `E2B_API_KEY`: API key for E2B sandbox environment
- `MODAL_TOKEN_ID`, `MODAL_TOKEN_SECRET`: Credentials for Modal serverless platform
- `TAVILY_API_KEY`: API key for Tavily search integration
- `EXA_API_KEY`: Credentials for Exa search service
- `PINECONE_API_KEY`: Authentication for Pinecone vector database
- `TPUF_API_KEY`: API key for TPUF archival memory service

**Service Endpoints:**
- `OPENAI_BASE_URL`: Override for OpenAI API endpoint (default: https://api.openai.com/v1)
- `AZURE_BASE_URL`: Azure OpenAI service endpoint
- `OLLAMA_BASE_URL`: Ollama server endpoint for local LLMs
- `VLLM_API_BASE`: vLLM inference server endpoint
- `LMSTUDIO_BASE_URL`: LMStudio server endpoint
- `GEMINI_BASE_URL`: Google Gemini API endpoint (default: https://generativelanguage.googleapis.com/)

These environment variables can be set in a `.env` file, exported in the shell environment, or passed directly to the application at runtime. The configuration system automatically detects and applies these values when initializing services and connections.

**Section sources**
- [settings.py](file://letta/settings.py#L114-L194)

## Settings Hierarchy and Override Mechanisms
Letta implements a multi-layered configuration hierarchy that determines how settings are resolved and applied. The system follows a specific precedence order, where values from higher-priority sources override those from lower-priority sources. This hierarchy ensures consistent behavior across different deployment scenarios while allowing for environment-specific customization.

The configuration hierarchy, from highest to lowest precedence, is as follows:

1. **Runtime Environment Variables**: Directly set environment variables have the highest priority and will override all other configuration sources. These are typically set in the deployment environment or shell session.

2. **Configuration File Settings**: The `~/.letta/config` file stores persistent configuration options that apply across sessions. This file is created and managed through the `letta configure` command and contains settings for storage backends, default models, and persona configurations.

3. **Database-Stored Configuration**: For multi-user deployments, organization-level and agent-specific settings can be stored in the database. This includes per-agent environment variables, sandbox configurations, and custom tool settings.

4. **Default Values**: When no explicit configuration is provided, Letta falls back to built-in default values defined in the codebase.

The system also supports dynamic configuration overrides through the API and CLI. For example, when creating or updating an agent, specific LLM or embedding configurations can be provided that override the global defaults for that particular agent. Similarly, tool execution can be configured with temporary environment variables that apply only to the duration of the tool's execution.

Configuration loading follows a specific process:
1. The system first checks for the `MEMGPT_CONFIG_PATH` environment variable to determine the configuration file location
2. If not specified, it defaults to `~/.letta/config`
3. The configuration file is parsed and loaded into memory
4. Environment variables are then applied, overriding any conflicting values from the configuration file
5. Finally, database-stored settings (for agent-specific or organization-level configurations) are applied

This hierarchical approach allows for flexible deployment scenarios, from single-user development environments to multi-tenant production systems, while maintaining security and consistency.

**Section sources**
- [config.py](file://letta/config.py#L97-L195)
- [settings.py](file://letta/settings.py#L230-L460)

## LLM Configuration Options
Letta provides extensive configuration options for language models, allowing fine-tuning of generation parameters, model selection, and provider-specific settings. The LLM configuration is managed through the `LLMConfig` class, which defines all parameters that control how the language model behaves during inference.

### Core Generation Parameters
- **Temperature**: Controls the randomness of the output (default: 0.7). Lower values produce more deterministic responses, while higher values increase creativity and randomness.
- **Max Tokens**: Sets the maximum number of tokens to generate in the response. If not specified, defaults vary by model (e.g., 16,384 for GPT-5, 8,192 for GPT-4.1).
- **Context Window**: Defines the maximum number of tokens the model can process in a single request. Default values are model-specific (e.g., 272,000 for GPT-5, 128,000 for GPT-4o).
- **Frequency Penalty**: Penalizes new tokens based on their existing frequency in the text, reducing repetition (range: -2.0 to 2.0).

### Model-Specific Configuration
Different LLM providers support unique configuration options:

**OpenAI Models:**
- **Reasoning Effort**: Controls the depth of reasoning for models like GPT-5 (options: none, minimal, low, medium, high)
- **Verbosity**: Soft control for output verbosity in GPT-5 models (low, medium, high)
- **Tier**: Cost tier selection for cloud-based models

**Anthropic Models:**
- **Enable Reasoner**: Toggles extended thinking capability for Claude models
- **Max Reasoning Tokens**: Configurable thinking budget for reasoning models (minimum: 1,024)
- **Effort**: Controls token spending for Claude Opus 4.5 (low, medium, high)

**Google Gemini Models:**
- **Thinking Budget**: Configurable budget for extended thinking in Gemini 2.5 Flash/Pro models
- **Force Minimum Thinking Budget**: Ensures a minimum thinking budget is applied

### Model Endpoint Configuration
The system supports multiple endpoint types for LLMs:
- **OpenAI**: Standard OpenAI API endpoints
- **Anthropic**: Anthropic Claude models
- **Google AI/Vertex**: Google's Gemini models through AI Studio or Vertex AI
- **Azure**: Azure OpenAI Service
- **Groq**: Groq inference platform
- **Ollama**: Local LLM server
- **vLLM**: High-performance inference server
- **Bedrock**: AWS Bedrock service
- **Local LLMs**: Various local inference servers (llama.cpp, KoboldCpp, etc.)

The `LLMConfig` class automatically applies model-specific defaults when a configuration is created. For example, when configuring a GPT-5 model, the context window is automatically set to 272,000 tokens, and the max_tokens parameter defaults to 16,384. Similarly, Claude models have reasoning capabilities enabled by default.

Configuration can be created programmatically using the `LLMConfig.default_config()` method, which provides pre-configured settings for commonly used models like "gpt-4", "gpt-4o-mini", and "gpt-4o".

**Section sources**
- [llm_config.py](file://letta/schemas/llm_config.py#L16-L528)

## Embedding Configuration
The embedding configuration system in Letta manages parameters for text embedding models used in retrieval-augmented generation (RAG) and memory operations. The `EmbeddingConfig` class defines all settings related to embedding model connections, processing parameters, and provider-specific configurations.

### Core Embedding Parameters
- **Embedding Model**: Specifies the embedding model to use (e.g., "text-embedding-ada-002", "text-embedding-3-small")
- **Embedding Dimension**: Defines the dimensionality of the embedding vectors (e.g., 1536 for text-embedding-ada-002, 2000 for text-embedding-3-small)
- **Embedding Chunk Size**: Controls the size of text chunks before embedding (default: 300 characters)
- **Batch Size**: Maximum number of embeddings to process in a single batch (default: 32)

### Supported Embedding Providers
Letta supports multiple embedding providers through different endpoint types:

**Cloud Providers:**
- **OpenAI**: text-embedding models through the OpenAI API
- **Anthropic**: Embedding capabilities through Anthropic's API
- **Google AI/Vertex**: Google's embedding models through AI Studio or Vertex AI
- **Azure**: Azure OpenAI embedding services
- **Together**: Embedding models through Together AI
- **Bedrock**: AWS Bedrock embedding services

**Local/Alternative Providers:**
- **Ollama**: Local embedding models through Ollama server
- **vLLM**: High-performance local embedding inference
- **Pinecone**: Direct integration with Pinecone's vector database embedding capabilities
- **Hugging Face**: Integration with Hugging Face models

### Provider-Specific Configuration
Certain providers require additional configuration parameters:

**Azure:**
- `azure_endpoint`: Azure OpenAI service endpoint
- `azure_version`: API version for the Azure service
- `azure_deployment`: Specific deployment name for the embedding model

The system provides convenience methods for creating default configurations for common embedding models. For example, `EmbeddingConfig.default_config()` can create pre-configured settings for models like "text-embedding-ada-002" with OpenAI, "text-embedding-3-small" with OpenAI, or the Letta-hosted embedding service.

When no explicit configuration is provided, the system defaults to using "text-embedding-3-small" with OpenAI as the embedding model, with a dimensionality of 2000 and the default chunk size of 300 characters.

**Section sources**
- [embedding_config.py](file://letta/schemas/embedding_config.py#L8-L88)

## Agent-Specific Settings
Letta supports granular configuration at the agent level, allowing individual agents to have customized behavior, tools, and execution parameters. This enables specialized agents with different capabilities within the same system.

### Agent Environment Variables
Agents can have their own set of environment variables that are isolated from other agents and the global configuration. These variables are stored in the database and can be managed through the API. The `AgentEnvironmentVariable` schema defines the structure for these variables:

- **Key**: The name of the environment variable
- **Value**: The value of the environment variable (can be encrypted)
- **Description**: Optional description of the variable's purpose
- **Agent ID**: Reference to the agent that owns the variable

These environment variables are automatically injected into the agent's execution context when the agent runs, allowing for agent-specific API keys, service endpoints, or configuration parameters.

### Agent Configuration Hierarchy
Agent settings follow a specific override hierarchy:
1. **Agent-Specific Settings**: Configuration applied directly to the agent takes highest precedence
2. **Organization-Level Settings**: Default settings for all agents within an organization
3. **Global Defaults**: System-wide default values

This allows organizations to establish baseline configurations while permitting individual agents to deviate when necessary.

### Agent Creation and Management
When creating agents, configuration options can be specified to customize their behavior:
- **LLM Configuration**: Specific LLM settings for the agent, overriding global defaults
- **Embedding Configuration**: Custom embedding model and parameters
- **Memory Configuration**: Initial memory state and block structure
- **Tool Selection**: Specific tools to be available to the agent
- **Personality and Persona**: Initial persona and human context

The agent configuration system also supports template-based creation, where predefined configurations can be reused across multiple agents, ensuring consistency while reducing configuration overhead.

**Section sources**
- [environment_variables.py](file://letta/schemas/environment_variables.py#L72-L88)
- [config.py](file://letta/config.py#L44-L50)

## Memory Configuration
Letta's memory system is highly configurable, allowing for customization of how agents store and retrieve information. The memory configuration determines the structure, capacity, and behavior of an agent's memory subsystem.

### Core Memory Parameters
- **Core Memory Persona Character Limit**: Maximum number of characters allowed in the persona memory block (default: 2000)
- **Core Memory Human Character Limit**: Maximum number of characters allowed in the human memory block (default: 2000)
- **Archival Memory Token Limit**: Maximum number of tokens allowed in archival memory (default: 8192)

### Memory Storage Configuration
The system supports different storage backends for various types of memory:

**Archival Storage:**
- **Type**: Storage backend type (sqlite, postgres, or external vector DB)
- **Path**: File system path for local storage
- **URI**: Connection URI for external databases

**Recall Storage:**
- **Type**: Storage backend type (sqlite, postgres, or external vector DB)
- **Path**: File system path for local storage
- **URI**: Connection URI for external databases

**Metadata Storage:**
- **Type**: Storage backend for metadata (sqlite, postgres)
- **Path**: File system path for local storage
- **URI**: Connection URI for external databases

### Memory Management Settings
Additional configuration options control memory behavior:
- **Summarization Mode**: Determines how memory is summarized when approaching capacity (STATIC_MESSAGE_BUFFER, PARTIAL_EVICT_MESSAGE_BUFFER)
- **Message Buffer Limit**: Maximum number of messages to keep in the buffer before summarization (default: 60)
- **Message Buffer Minimum**: Minimum number of messages to keep after summarization (default: 15)
- **Partial Evict Summarizer Percentage**: Percentage of messages to evict during partial eviction (default: 0.30)

The memory system also supports file-based memory, where agents can maintain memory of specific files with configurable limits:
- **Max Files Open**: Maximum number of files that can be open in memory simultaneously
- **File Block Limits**: Character limits for individual file memory blocks

These memory configuration options can be set globally through the main configuration file or overridden at the agent level, allowing for agents with different memory capacities and behaviors based on their specific use cases.

**Section sources**
- [config.py](file://letta/config.py#L60-L89)
- [memory.py](file://letta/schemas/memory.py#L1-L467)
- [summarizer_settings.py](file://letta/settings.py#L60-L98)

## Tool Selection and Execution Parameters
Letta's tool system provides extensive configuration options for managing external tool integration, execution parameters, and security settings. The configuration system allows for fine-grained control over how tools are selected, executed, and monitored.

### Tool Configuration Options
Each tool can be configured with the following parameters:
- **Return Character Limit**: Maximum number of characters in the tool's response (default: 5,000, configurable up to 1,000,000)
- **Pip Requirements**: Python packages required by the tool
- **Npm Requirements**: Node.js packages required by the tool
- **Default Requires Approval**: Whether tool execution requires user approval by default
- **Enable Parallel Execution**: Whether the tool can be executed concurrently with other tools

### Tool Execution Environment
Tools can be executed in different sandbox environments, each with its own configuration:

**E2B Sandbox:**
- **E2B API Key**: Authentication for E2B sandbox service
- **E2B Sandbox Template ID**: Pre-configured template for sandbox instances

**Modal Sandbox:**
- **Modal Token ID**: Authentication identifier for Modal
- **Modal Token Secret**: Authentication secret for Modal

**Local Sandbox:**
- **Tool Execution Directory**: File system directory for tool execution
- **Tool Execution Timeout**: Maximum execution time in seconds (default: 180)
- **Tool Execution Virtual Environment**: Python virtual environment for tool execution
- **Tool Execution Auto-reload Environment**: Whether to automatically reload the virtual environment

### Tool Management Configuration
The system supports various tool sources and management options:
- **Built-in Tools**: Core functionality provided by Letta (file operations, memory management, etc.)
- **Custom Tools**: User-defined tools with custom code
- **MCP Tools**: Tools provided through the Model Context Protocol
- **Composio Tools**: Integration with Composio's tool library

Tool discovery and management can be configured through:
- **MCP Configuration File**: JSON file listing available MCP servers
- **Dynamic Tool Registration**: Runtime registration of new tools
- **Tool Caching**: Configuration of tool metadata caching

The tool execution system also supports environment variable injection, allowing tools to access specific credentials or configuration parameters without exposing them globally.

**Section sources**
- [tool.py](file://letta/schemas/tool.py#L31-L209)
- [tool_settings.py](file://letta/settings.py#L17-L41)
- [sandbox_config_manager.py](file://letta/services/sandbox_config_manager.py#L28-L47)

## Security Considerations for Sensitive Credentials
Letta implements robust security measures for managing sensitive credentials and configuration data. The system follows security best practices to protect API keys, database credentials, and other sensitive information.

### Credential Encryption
All sensitive credentials are encrypted at rest using AES-256-GCM encryption:
- **Encryption Key**: Managed through the `LETTA_ENCRYPTION_KEY` environment variable
- **Key Derivation**: PBKDF2 with SHA-256 and 100,000 iterations
- **Salt Generation**: Random 16-byte salt for each encryption operation
- **IV Generation**: Random 12-byte initialization vector

The `Secret` class provides a secure wrapper for sensitive values, ensuring they remain encrypted in memory whenever possible. The class implements:
- **Lazy Decryption**: Values are only decrypted when absolutely necessary
- **Value Caching**: Decrypted values are cached to avoid repeated decryption
- **Memory Protection**: Attempts to prevent sensitive data from being written to swap or crash dumps

### Secure Credential Storage
The system uses a dual-write approach during the transition to encrypted storage:
- **Encrypted Column**: Stores the encrypted value (primary storage)
- **Plaintext Column**: Stores the plaintext value (temporary, for migration)
- **Automatic Migration**: System gradually migrates from plaintext to encrypted storage

When accessing credentials, the system prioritizes the encrypted value and only falls back to plaintext during the migration period.

### Environment Variable Security
The configuration system validates and sanitizes environment variables:
- **Secure Defaults**: Reasonable defaults for security-related settings
- **Input Validation**: Validation of configuration values to prevent injection attacks
- **Secure Transmission**: Environment variables are not logged or exposed in error messages

### Security Best Practices
Recommended security practices include:
- **Encryption Key Management**: Store the encryption key in a secure secrets manager, not in version control
- **Least Privilege**: Grant tools and services only the minimum permissions they need
- **Regular Rotation**: Regularly rotate API keys and credentials
- **Environment Separation**: Use different credentials for development, staging, and production
- **Audit Logging**: Enable logging to monitor access to sensitive configuration

The system also supports security-focused configuration options:
- **No Default Actor**: Prevents fallback to default actor when actor_id is None
- **Disable Default Actor Fallback**: Raises errors instead of using default actors
- **Secure Token Management**: Tokens are never stored in plaintext when possible

**Section sources**
- [secret.py](file://letta/schemas/secret.py#L1-L192)
- [crypto_utils.py](file://letta/helpers/crypto_utils.py#L1-L41)
- [settings.py](file://letta/settings.py#L339-L340)

## Environment Separation
Letta supports multiple deployment environments through configurable settings that adapt behavior based on the current environment. This allows for safe development, testing, and production workflows with appropriate isolation and security controls.

### Environment Configuration
The system identifies the current environment through the `environment` setting, which can be set to:
- **PRODUCTION**: Production environment with strict security and performance settings
- **DEV**: Development environment with debugging enabled
- **STAGING**: Staging environment for pre-production testing

### Environment-Specific Settings
Different environments have different default configurations:

**Development Environment:**
- Debug logging enabled
- Hot-reload for code changes
- Less restrictive security settings
- Local storage backends
- Detailed error messages

**Staging Environment:**
- Moderate logging levels
- Performance monitoring enabled
- Security settings close to production
- Database-backed storage
- Limited error details

**Production Environment:**
- Minimal logging for performance
- Security hardening
- Database-backed storage with connection pooling
- Error messages sanitized to prevent information disclosure
- Performance optimizations

### Configuration Management
Environment separation is managed through:
- **Environment Variables**: Different `.env` files for each environment
- **Configuration Files**: Environment-specific configuration files
- **Database Settings**: Environment-specific settings stored in the database
- **CLI Parameters**: Command-line options that override environment settings

The system also supports environment-specific feature flags:
- **Experimental Features**: Enabled only in development environments
- **Performance Monitoring**: Enabled in staging and production
- **Telemetry Collection**: Configurable by environment
- **Security Controls**: Stricter in production

### Deployment Best Practices
Recommended practices for environment separation include:
- **Separate Databases**: Use different database instances for each environment
- **Isolated Credentials**: Different API keys and credentials for each environment
- **Configuration Validation**: Validate configuration before deployment
- **Environment Variables Management**: Use environment-specific `.env` files or secrets managers
- **Automated Deployment**: Use deployment scripts that apply the correct configuration for each environment

This approach ensures that changes can be safely tested in development and staging environments before being deployed to production, with appropriate security and performance characteristics for each environment.

**Section sources**
- [settings.py](file://letta/settings.py#L236-L237)

## Performance Tuning
Letta provides several configuration options for performance tuning, allowing optimization of resource usage, response times, and system throughput. These settings can be adjusted based on the deployment environment, hardware capabilities, and performance requirements.

### Database Performance
Database-related performance settings include:
- **Connection Pooling**: Configurable connection pool size (default: 25), overflow limit (default: 10), and timeout (default: 30 seconds)
- **Connection Recycling**: Automatic recycling of connections after 1800 seconds
- **Pre-Ping**: Verification of connection validity before use
- **LIFO Pooling**: Use last-in, first-out connection allocation for better connection reuse

### Request and Timeout Settings
- **LLM Request Timeout**: Global timeout for LLM requests (default: 60 seconds, configurable between 10-1800 seconds)
- **LLM Stream Timeout**: Timeout for streaming LLM responses (default: 60 seconds)
- **File Processing Timeout**: Maximum time for file processing operations (default: 30 minutes)
- **Keepalive Interval**: Interval for SSE keepalive messages to prevent timeouts (default: 50 seconds)

### Caching and Optimization
- **Database Pool Monitoring**: Enabled by default, collects connection pool statistics every 30 seconds
- **Event Loop Parallelism**: Configurable thread pool size for event loop operations (default: 43 workers)
- **Batch Job Polling**: Configurable interval for polling running LLM batches (default: 5 minutes)
- **Cron Job Parameters**: Configurable lookback period (default: 2 weeks) and batch size for job polling

### Resource Management
- **UVicorn Workers**: Number of worker processes for the FastAPI server (default: 1)
- **UVicorn Timeout**: Keep-alive timeout for UVicorn (default: 5 seconds)
- **Use UVLoop**: Option to use uvloop as the asyncio event loop for better performance
- **Use Granian**: Option to use Granian for workers instead of standard workers

### Telemetry and Monitoring
Performance monitoring can be configured through:
- **OTEL Exporter Endpoint**: OpenTelemetry collector endpoint for metrics and traces
- **Tracing Configuration**: Enable or disable OTEL tracing
- **LLM API Logging**: Enable detailed logging of LLM API calls
- **Stop Reason Tracking**: Enable tracking of step completion reasons
- **Provider Trace Tracking**: Enable tracking of raw LLM requests and responses

### Optimization Recommendations
For optimal performance:
- **Production**: Increase connection pool size and enable connection pooling
- **High-Load**: Increase UVicorn workers and adjust event loop parallelism
- **Low-Latency**: Reduce timeouts and enable keepalive messages
- **Resource-Constrained**: Disable non-essential telemetry and monitoring
- **Batch Processing**: Adjust batch job polling parameters for workload patterns

These performance settings can be tuned based on the specific requirements of the deployment, balancing responsiveness, resource usage, and reliability.

**Section sources**
- [settings.py](file://letta/settings.py#L256-L317)

## Configuration Change Monitoring
Letta includes comprehensive monitoring and tracking capabilities for configuration changes, enabling audit trails, performance analysis, and system observability. The configuration system integrates with telemetry and monitoring tools to provide visibility into how settings affect system behavior.

### Configuration Tracking
The system tracks various configuration-related metrics:
- **Agent Run Tracking**: Monitor agent runs with cancellation support
- **Errored Messages**: Track messages that resulted in errors
- **Stop Reason Tracking**: Monitor reasons for step completion
- **Provider Trace Tracking**: Track raw LLM requests and responses

### Telemetry Configuration
Telemetry settings can be configured to control the level of monitoring:
- **Disable Tracing**: Option to disable OTEL tracing for performance
- **LLM API Logging**: Enable detailed logging of LLM API interactions
- **Track Last Agent Run**: Update metrics for the last agent run
- **Track Provider Trace**: Enable tracking of raw LLM requests and responses

### Observability Integration
The system supports integration with observability platforms:
- **OpenTelemetry**: Export metrics and traces to OTEL collectors
- **Datadog**: Support for Datadog APM and profiling
- **ClickHouse**: Configuration for ClickHouse-based observability
- **Signoz**: Support for Signoz observability platform

### Performance Metrics
Key performance metrics tracked include:
- **LLM Execution Time**: Histogram of LLM execution times
- **Tool Execution Time**: Histogram of tool execution times
- **Step Execution Time**: Histogram of step execution times
- **Time to First Token**: Histogram of time to first token in streaming responses
- **Database Connection Pool Metrics**: Connection counts, availability, and duration

### Monitoring Best Practices
Recommended monitoring practices include:
- **Enable Telemetry in Production**: Collect metrics for performance analysis
- **Set Appropriate Timeouts**: Configure timeouts based on expected response times
- **Monitor Connection Pools**: Watch for connection pool exhaustion
- **Track Error Rates**: Monitor for increases in errored messages
- **Analyze Stop Reasons**: Understand why steps are completing
- **Review Provider Traces**: Analyze raw LLM interactions for optimization

The configuration change monitoring system provides valuable insights into system behavior, helping administrators optimize performance, troubleshoot issues, and ensure reliable operation across different deployment environments.

**Section sources**
- [settings.py](file://letta/settings.py#L277-L288)
- [metric_registry.py](file://letta/otel/metric_registry.py#L44-L275)