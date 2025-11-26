# Getting Started

<cite>
**Referenced Files in This Document**   
- [README.md](file://README.md)
- [pyproject.toml](file://pyproject.toml)
- [compose.yaml](file://compose.yaml)
- [dev-compose.yaml](file://dev-compose.yaml)
- [Dockerfile](file://Dockerfile)
- [init.sql](file://init.sql)
- [config.py](file://letta/config.py)
- [settings.py](file://letta/settings.py)
- [main.py](file://letta/main.py)
- [startup.sh](file://letta/server/startup.sh)
- [nginx.conf](file://nginx.conf)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Installation with Poetry](#installation-with-poetry)
4. [Environment Configuration](#environment-configuration)
5. [Docker Compose Setup](#docker-compose-setup)
6. [Development vs Production Configuration](#development-vs-production-configuration)
7. [Service Initialization](#service-initialization)
8. [Basic Agent Operations](#basic-agent-operations)
9. [Configuration Options](#configuration-options)
10. [Troubleshooting](#troubleshooting)
11. [Platform-Specific Considerations](#platform-specific-considerations)

## Introduction

This guide provides comprehensive instructions for setting up the Letta development environment. Letta is a platform for building stateful AI agents with advanced memory capabilities that can learn and self-improve over time. The setup process involves installing dependencies, configuring environment variables, launching services via Docker Compose, and verifying the installation with basic agent operations.

The Letta architecture consists of several core components:
- **API Server**: The main service that handles agent operations and API requests
- **PostgreSQL Database**: Stores agent state, memory, and metadata with pgvector extension for vector operations
- **Nginx Reverse Proxy**: Handles API routing and load balancing
- **OpenTelemetry Collector**: Manages observability and monitoring data

This guide will walk you through the complete setup process, from dependency installation to running your first "Hello World" agent example.

**Section sources**
- [README.md](file://README.md#L1-L523)

## Prerequisites

Before beginning the installation process, ensure your system meets the following requirements:

### System Requirements
- **Operating System**: Windows 10/11, macOS 10.15+, or Linux (Ubuntu 20.04+ recommended)
- **Python**: Version 3.11 or higher
- **Docker**: Version 20.10 or higher with Docker Compose v2
- **Disk Space**: At least 2GB of free space for containers and dependencies
- **Memory**: Minimum 4GB RAM (8GB recommended for optimal performance)

### Required Tools
- **Poetry**: Python dependency management tool (version 1.6+)
- **uv**: Fast Python package installer and resolver
- **Git**: For cloning the repository (if installing from source)

The installation process uses Poetry as the primary dependency manager, which provides deterministic dependency resolution and isolated virtual environments. Docker Compose is used to orchestrate the multi-container application, ensuring consistent deployment across different environments.

**Section sources**
- [pyproject.toml](file://pyproject.toml#L1-L206)
- [README.md](file://README.md#L501-L510)

## Installation with Poetry

To install Letta dependencies using Poetry, follow these steps:

1. **Clone the repository** (if not already cloned):
```bash
git clone https://github.com/letta-ai/letta.git
cd letta
```

2. **Install dependencies with Poetry**:
```bash
poetry install --all-extras
```

This command installs all required dependencies including:
- Core framework dependencies (FastAPI, SQLAlchemy, Pydantic)
- Database connectors (PostgreSQL, Redis, SQLite)
- LLM providers (OpenAI, Anthropic, Bedrock, etc.)
- Development tools (pytest, pre-commit, pyright)

3. **Alternative installation using uv** (faster):
```bash
uv sync --all-extras
```

The `--all-extras` flag ensures all optional dependencies are installed, including database connectors, server components, and development tools. This comprehensive installation supports all Letta features and is recommended for development environments.

The dependencies are defined in `pyproject.toml` and include specific versions to ensure compatibility. Key dependencies include:
- FastAPI for the REST API server
- SQLAlchemy with asyncio support for database operations
- Pydantic for data validation and settings management
- Various LLM client libraries (OpenAI, Anthropic, etc.)

**Section sources**
- [pyproject.toml](file://pyproject.toml#L1-L206)
- [README.md](file://README.md#L501-L510)

## Environment Configuration

Proper environment configuration is essential for Letta to function correctly. The system uses environment variables to configure various components.

### Required Environment Variables
Create a `.env` file in the project root directory with the following essential variables:

```env
# Database Configuration
LETTA_PG_USER=letta
LETTA_PG_PASSWORD=letta
LETTA_PG_DB=letta
LETTA_PG_PORT=5432

# LLM Provider API Keys (choose one or more)
OPENAI_API_KEY=your_openai_key
ANTHROPIC_API_KEY=your_anthropic_key
GROQ_API_KEY=your_groq_key
GEMINI_API_KEY=your_gemini_key

# Optional Configuration
LETTA_DEBUG=True
```

### Additional Provider-Specific Variables
For other LLM providers, include the appropriate environment variables:

```env
# Azure Configuration
AZURE_API_KEY=your_azure_key
AZURE_BASE_URL=your_azure_endpoint
AZURE_API_VERSION=2024-09-01-preview

# Ollama Configuration
OLLAMA_BASE_URL=http://localhost:11434

# VLLM Configuration
VLLM_API_BASE=http://localhost:8000
```

### Environment Variable Reference
The system recognizes the following environment variables for configuration:

- **Database**: `LETTA_PG_USER`, `LETTA_PG_PASSWORD`, `LETTA_PG_DB`, `LETTA_PG_PORT`
- **LLM Providers**: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GROQ_API_KEY`, `GEMINI_API_KEY`
- **Server**: `LETTA_DEBUG`, `LETTA_OTEL_EXPORTER_OTLP_ENDPOINT`
- **Tool Execution**: `LETTA_SANDBOX_MOUNT_PATH`

These variables are automatically loaded by the application and used to configure the various services. The `.env` file is referenced in the `compose.yaml` file and loaded into the containers during startup.

**Section sources**
- [compose.yaml](file://compose.yaml#L32-L53)
- [settings.py](file://letta/settings.py#L114-L194)
- [config.py](file://letta/config.py#L6-L89)

## Docker Compose Setup

Letta uses Docker Compose to manage its multi-container architecture. Two primary compose files are provided for different use cases.

### Default Configuration (compose.yaml)
The default `compose.yaml` file defines three services:

```yaml
services:
  letta_db:
    image: ankane/pgvector:v0.5.1
    environment:
      - POSTGRES_USER=${LETTA_PG_USER:-letta}
      - POSTGRES_PASSWORD=${LETTA_PG_PASSWORD:-letta}
      - POSTGRES_DB=${LETTA_PG_DB:-letta}
    volumes:
      - ./.persist/pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "${LETTA_PG_PORT:-5432}:5432"
  
  letta_server:
    image: letta/letta:latest
    depends_on:
      letta_db:
        condition: service_healthy
    ports:
      - "8083:8083"
      - "8283:8283"
    env_file:
      - .env
  
  letta_nginx:
    image: nginx:stable-alpine3.17-slim
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "80:80"
```

### Development Configuration (dev-compose.yaml)
For development, use `dev-compose.yaml` which builds the server from source:

```yaml
services:
  letta_db:
    # Same as compose.yaml
  letta_server:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    # Additional development configuration
```

### Launching the Services
To start the Letta services:

1. Ensure Docker is running
2. Navigate to the project directory
3. Run the appropriate command:

```bash
# For production-like setup
docker compose -f compose.yaml up -d

# For development setup
docker compose -f dev-compose.yaml up -d
```

The `-d` flag runs the containers in detached mode. The services will start in the following order:
1. PostgreSQL database with pgvector extension
2. Letta API server (waits for database to be healthy)
3. Nginx reverse proxy

**Section sources**
- [compose.yaml](file://compose.yaml#L1-L66)
- [dev-compose.yaml](file://dev-compose.yaml#L1-L49)
- [Dockerfile](file://Dockerfile#L1-L89)

## Development vs Production Configuration

Letta provides different configuration options for development and production environments, allowing for flexibility in deployment scenarios.

### Development Setup
The development configuration (`dev-compose.yaml`) is designed for local development and testing:

- **Builds from source**: The server container is built from the local codebase
- **Hot reloading**: Code changes are automatically reflected in the running container
- **Verbose logging**: Debug mode is enabled by default
- **Local database**: Uses a persistent volume for data storage

Key characteristics:
- Uses `Dockerfile` with build context
- Exposes additional ports for debugging
- Includes development dependencies
- Mounts source code into the container

### Production Setup
The production configuration (`compose.yaml`) is optimized for stable deployment:

- **Pre-built image**: Uses the official `letta/letta:latest` image
- **Optimized performance**: Production-grade configurations
- **Security considerations**: Environment variables are properly isolated
- **Scalability**: Designed for container orchestration

Key differences:
- Uses pre-built Docker image from Docker Hub
- No source code mounting (immutable containers)
- Optimized resource allocation
- Production-focused environment variables

### Configuration Comparison
| Aspect | Development | Production |
|--------|-------------|------------|
| Image Source | Built from local Dockerfile | Pre-built from Docker Hub |
| Code Updates | Hot reload supported | Requires rebuild |
| Logging | Verbose (debug mode) | Standard (production mode) |
| Database | Local persistent volume | External or managed database |
| Dependencies | All extras included | Minimal required packages |

The choice between development and production configurations depends on your use case. Use development mode when modifying the Letta source code, and production mode for stable deployments.

**Section sources**
- [compose.yaml](file://compose.yaml#L1-L66)
- [dev-compose.yaml](file://dev-compose.yaml#L1-L49)
- [Dockerfile](file://Dockerfile#L1-L89)

## Service Initialization

The service initialization process involves several steps to ensure all components are properly configured and running.

### Startup Sequence
When launching Letta with Docker Compose, the following sequence occurs:

1. **Database Initialization**: The PostgreSQL container starts and runs the initialization script
2. **Health Check**: The database container performs health checks to ensure readiness
3. **Server Startup**: The Letta server container starts after the database is healthy
4. **Database Migration**: Alembic runs database migrations to set up the schema
5. **Service Registration**: The API server registers endpoints and starts listening

### Database Initialization
The `init.sql` script configures the PostgreSQL database with Letta-specific settings:

```sql
-- Set up schema and extensions
\c :"db_name"

CREATE SCHEMA :"db_name"
    AUTHORIZATION :"db_user";

ALTER DATABASE :"db_name"
    SET search_path TO :"db_name";

CREATE EXTENSION IF NOT EXISTS vector WITH SCHEMA :"db_name";
DROP SCHEMA IF EXISTS public CASCADE;
```

This script:
- Creates a dedicated schema for Letta
- Sets the search path to the Letta schema
- Installs the pgvector extension for vector operations
- Removes the public schema for security

### Server Startup Process
The `startup.sh` script orchestrates the server initialization:

```bash
# Wait for PostgreSQL to be ready
wait_for_postgres

# Run database migrations
alembic upgrade head

# Start OpenTelemetry Collector
/usr/local/bin/otelcol-contrib --config "$CONFIG_FILE" &

# Start Letta server
exec letta server --host $HOST --port $PORT
```

The script handles:
- Database connectivity verification
- Automatic schema migration
- OpenTelemetry collector startup
- Proper process management with cleanup traps

### Verification Steps
After startup, verify the services are running:

```bash
# Check container status
docker compose ps

# View logs
docker compose logs letta_server

# Test API connectivity
curl http://localhost:8283/health
```

Successful initialization will show all containers in "running" state and return a 200 status from the health endpoint.

**Section sources**
- [init.sql](file://init.sql#L1-L37)
- [startup.sh](file://letta/server/startup.sh#L1-L82)
- [Dockerfile](file://Dockerfile#L1-L89)

## Basic Agent Operations

Once the Letta environment is set up, you can perform basic agent operations to verify functionality.

### Agent Creation
Create a simple agent using the API:

```python
from letta_client import Letta
import os

client = Letta(token=os.getenv("LETTA_API_KEY"))
# For self-hosting: client = Letta(base_url="http://localhost:8283")

agent_state = client.agents.create(
    model="openai/gpt-4.1",
    memory_blocks=[
        {
            "label": "human",
            "value": "The human's name is Chad. They like vibe coding."
        },
        {
            "label": "persona",
            "value": "My name is Sam, a helpful assistant."
        }
    ],
    tools=["web_search", "run_code"]
)

print(f"Created agent with ID: {agent_state.id}")
```

### Sending Messages
Interact with the agent by sending messages:

```python
response = client.agents.messages.create(
    agent_id=agent_state.id,
    messages=[
        {
            "role": "user",
            "content": "Hey, nice to meet you, my name is Brad."
        }
    ]
)

for message in response.messages:
    print(message)
```

### Tool Attachment
Attach tools to existing agents:

```python
# Add MCP tool
tool = client.tools.add_mcp_tool(
    mcp_server_name="weather-server",
    mcp_tool_name="get_weather"
)

# Attach tool to agent
client.agents.tool.attach(
    agent_id=agent_state.id,
    tool_id=tool.id
)
```

### Hello World Example
Verify the installation with a simple "Hello World" agent:

```python
# Create a basic agent
agent = client.agents.create(
    model="openai/gpt-4o-mini",
    memory_blocks=[
        {"label": "persona", "value": "You are a helpful assistant."}
    ]
)

# Send a test message
response = client.agents.messages.create(
    agent_id=agent.id,
    messages=[{"role": "user", "content": "Say hello world"}]
)

# Print the response
print(response.messages[0].content)
```

These operations demonstrate the core functionality of Letta and verify that the installation is working correctly.

**Section sources**
- [README.md](file://README.md#L36-L83)
- [main.py](file://letta/main.py#L1-L19)
- [settings.py](file://letta/settings.py#L196-L209)

## Configuration Options

Letta provides extensive configuration options through both environment variables and configuration files.

### Default Configuration
The default configuration is defined in `config.py` and includes:

```python
@dataclass
class LettaConfig:
    # Default presets
    preset: str = DEFAULT_PRESET
    persona: str = DEFAULT_PERSONA
    human: str = DEFAULT_HUMAN
    
    # Storage configurations
    archival_storage_type: str = "sqlite"
    recall_storage_type: str = "sqlite"
    metadata_storage_type: str = "sqlite"
    
    # Memory limits
    core_memory_persona_char_limit: int = CORE_MEMORY_PERSONA_CHAR_LIMIT
    core_memory_human_char_limit: int = CORE_MEMORY_HUMAN_CHAR_LIMIT
```

### Customizable Settings
Key configurable options include:

**Database Settings:**
- `LETTA_PG_URI`: Full PostgreSQL connection URI
- `LETTA_PG_USER`, `LETTA_PG_PASSWORD`, `LETTA_PG_DB`: Individual database credentials
- `LETTA_PG_PORT`: Database port (default: 5432)

**Server Settings:**
- `LETTA_DEBUG`: Enable debug mode (default: False)
- `HOST`: Server host (default: 0.0.0.0)
- `PORT`: Server port (default: 8283)

**LLM Provider Settings:**
- `OPENAI_API_KEY`: OpenAI API key
- `ANTHROPIC_API_KEY`: Anthropic API key
- `OLLAMA_BASE_URL`: Ollama server URL
- `VLLM_API_BASE`: vLLM server URL

### Configuration Hierarchy
Letta follows a specific configuration hierarchy:

1. **Environment Variables**: Highest priority, override all other settings
2. **Configuration Files**: Stored in `~/.letta/config`
3. **Default Values**: Hardcoded defaults in the source code

The `settings.py` file uses Pydantic's `BaseSettings` with environment variable support, allowing for flexible configuration through multiple methods.

### Advanced Configuration
For advanced use cases, additional configuration options are available:

```env
# Telemetry and Monitoring
LETTA_OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
CLICKHOUSE_ENDPOINT=your_clickhouse_url
CLICKHOUSE_PASSWORD=your_password

# Tool Execution
LETTA_SANDBOX_MOUNT_PATH=/path/to/tool/execution/dir

# Security
ENCRYPTION_KEY=your_encryption_key
```

These options enable features like observability integration, custom tool execution environments, and data encryption.

**Section sources**
- [config.py](file://letta/config.py#L1-L311)
- [settings.py](file://letta/settings.py#L1-L460)
- [compose.yaml](file://compose.yaml#L35-L53)

## Troubleshooting

This section addresses common issues encountered during Letta setup and provides solutions.

### Database Initialization Issues
**Problem**: Database container fails to start or migrations fail
**Symptoms**: 
- "Waiting for PostgreSQL to be ready..." message repeats
- Alembic migration errors
- Container restarts repeatedly

**Solutions**:
1. Ensure sufficient disk space is available
2. Check if port 5432 is already in use:
   ```bash
   lsof -i :5432  # macOS/Linux
   netstat -ano | findstr :5432  # Windows
   ```
3. Clear persistent data if corrupted:
   ```bash
   rm -rf ./.persist/pgdata
   docker compose -f compose.yaml down
   docker compose -f compose.yaml up -d
   ```

### Port Conflicts
**Problem**: Service fails to start due to port conflicts
**Common Conflicts**:
- Port 5432: PostgreSQL
- Port 8283: Letta API server
- Port 80: Nginx reverse proxy

**Solutions**:
1. Change ports in environment variables:
   ```env
   LETTA_PG_PORT=5433
   ```
2. Or modify `compose.yaml`:
   ```yaml
   ports:
     - "5433:5432"  # Map host port 5433 to container 5432
   ```

### Dependency Resolution Issues
**Problem**: Poetry or uv fails to install dependencies
**Solutions**:
1. Clear the package cache:
   ```bash
   poetry cache clear pypi --all
   uv cache clean
   ```
2. Update Poetry and uv to latest versions
3. Ensure Python 3.11+ is installed and active

### Container Startup Failures
**Problem**: Containers start but immediately exit
**Diagnosis**:
```bash
# Check container logs
docker compose logs letta_server
docker compose logs letta_db

# Check container status
docker compose ps
```

**Common Fixes**:
1. Ensure `.env` file has correct API keys
2. Verify database credentials match in `.env` and `compose.yaml`
3. Check file permissions for mounted volumes

### API Connectivity Issues
**Problem**: Cannot connect to API endpoints
**Troubleshooting**:
1. Test basic connectivity:
   ```bash
   curl http://localhost:8283/health
   ```
2. Check if firewall is blocking ports
3. Verify Nginx configuration in `nginx.conf`
4. Ensure the server is listening on the correct interface

### Memory and Performance Issues
**Problem**: High memory usage or slow performance
**Optimizations**:
1. Adjust database connection pool settings in `settings.py`
2. Monitor resource usage with `docker stats`
3. Consider using external database for production

**Section sources**
- [startup.sh](file://letta/server/startup.sh#L1-L82)
- [settings.py](file://letta/settings.py#L249-L264)
- [nginx.conf](file://nginx.conf#L1-L29)

## Platform-Specific Considerations

Different operating systems have specific considerations for running Letta successfully.

### Windows
**Requirements**:
- Windows 10/11 with WSL2 (recommended) or Docker Desktop
- PowerShell or Command Prompt with administrator privileges

**Specific Issues**:
1. **File Path Separators**: Use forward slashes or double backslashes in paths
2. **Line Endings**: Ensure scripts use CRLF line endings
3. **Volume Mounting**: Windows paths in Docker Compose:
   ```yaml
   volumes:
     - C:/path/to/data:/container/path
   ```

**Windows-Specific Commands**:
```cmd
:: Set environment variables
set LETTA_PG_USER=letta

:: Check port usage
netstat -ano | findstr :5432

:: Clear DNS cache
ipconfig /flushdns
```

### macOS
**Requirements**:
- macOS 10.15+ with Intel or Apple Silicon
- Homebrew for package management (recommended)

**Specific Issues**:
1. **ARM64 Architecture**: Ensure Docker images support Apple Silicon
2. **File Permissions**: macOS security may restrict file access
3. **Memory Management**: Adjust Docker Desktop memory allocation

**macOS-Specific Commands**:
```bash
# Install dependencies with Homebrew
brew install poetry uv

# Check system architecture
uname -m

# Clear DNS cache
sudo dscacheutil -flushcache
```

### Linux
**Requirements**:
- Ubuntu 20.04+ or equivalent distribution
- systemd for service management (optional)

**Specific Issues**:
1. **User Permissions**: Ensure user is in docker group
2. **Firewall**: Configure ufw or iptables for port access
3. **Systemd Services**: Create systemd units for auto-start

**Linux-Specific Commands**:
```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Check available disk space
df -h

# Monitor system resources
htop
```

### Cross-Platform Best Practices
1. **Use Environment Variables**: Store configuration in `.env` files
2. **Consistent Line Endings**: Use LF (Unix) line endings in scripts
3. **Relative Paths**: Use relative paths in configuration files
4. **Resource Allocation**: Adjust Docker resource limits based on system capabilities

**Section sources**
- [Dockerfile](file://Dockerfile#L47-L59)
- [startup.sh](file://letta/server/startup.sh#L1-L82)
- [compose.yaml](file://compose.yaml#L1-L66)