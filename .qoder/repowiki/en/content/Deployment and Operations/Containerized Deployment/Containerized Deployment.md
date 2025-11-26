# Containerized Deployment

<cite>
**Referenced Files in This Document**   
- [Dockerfile](file://Dockerfile)
- [compose.yaml](file://compose.yaml)
- [dev-compose.yaml](file://dev-compose.yaml)
- [scripts/pack_docker.sh](file://scripts/pack_docker.sh)
- [init.sql](file://init.sql)
- [nginx.conf](file://nginx.conf)
- [scripts/wait_for_service.sh](file://scripts/wait_for_service.sh)
- [letta/server/startup.sh](file://letta/server/startup.sh)
- [pyproject.toml](file://pyproject.toml)
- [README.md](file://README.md)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Multi-Stage Docker Build Process](#multi-stage-docker-build-process)
3. [Production vs Development Deployment](#production-vs-development-deployment)
4. [Image Building and Publishing Process](#image-building-and-publishing-process)
5. [Deployment Configuration and Environment Setup](#deployment-configuration-and-environment-setup)
6. [Container Security and Best Practices](#container-security-and-best-practices)
7. [Common Deployment Issues and Solutions](#common-deployment-issues-and-solutions)
8. [Conclusion](#conclusion)

## Introduction
Letta provides a containerized deployment solution using Docker and Docker Compose for both production and development environments. The deployment architecture is designed to be flexible, scalable, and secure, with support for various deployment scenarios including local development, cloud deployment, and on-premise installations. This document details the containerization strategy, deployment configurations, and best practices for deploying Letta in different environments.

The containerized deployment system consists of multiple components including a multi-stage Docker build process, production and development Docker Compose configurations, automated image building and publishing scripts, and comprehensive environment management. The system is designed to ensure consistent behavior across different deployment environments while providing flexibility for customization and extension.

**Section sources**
- [README.md](file://README.md#L1-L200)

## Multi-Stage Docker Build Process

The Letta Docker build process follows a multi-stage approach with distinct builder and runtime stages to optimize image size and security. This approach separates the build-time dependencies from the runtime environment, resulting in a leaner and more secure production image.

### Builder Stage
The builder stage starts with the `ankane/pgvector:v0.5.1` base image, which provides PostgreSQL with vector database capabilities. This stage installs Python and essential build tools required for dependency resolution:

```dockerfile
FROM ankane/pgvector:v0.5.1 AS builder

RUN apt-get update && apt-get install -y \
    python3 \
    python3-venv \
    python3-full \
    build-essential \
    libpq-dev \
    python3-dev \
    && rm -rf /var/lib/apt/lists/*
```

The builder stage creates a virtual environment and installs `uv`, a fast Python package installer and resolver, which is used to install all dependencies specified in `pyproject.toml` and `uv.lock`. This ensures deterministic builds with pinned dependency versions:

```dockerfile
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

COPY pyproject.toml uv.lock ./
COPY . .

RUN uv sync --frozen --no-dev --all-extras --python 3.11
```

### Runtime Stage
The runtime stage also starts with the `ankane/pgvector:v0.5.1` base image but installs only the minimal dependencies required for production execution. This stage includes Node.js for server-side operations and the OpenTelemetry Collector for observability:

```dockerfile
FROM ankane/pgvector:v0.5.1 AS runtime

ARG NODE_VERSION=22

RUN apt-get update && \
    apt-get install -y curl python3 python3-venv libpq-dev && \
    curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION}.x | bash - && \
    apt-get install -y nodejs && \
    curl -L https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.96.0/otelcol-contrib_0.96.0_linux_amd64.tar.gz -o /tmp/otel-collector.tar.gz && \
    tar xzf /tmp/otel-collector.tar.gz -C /usr/local/bin && \
    rm /tmp/otel-collector.tar.gz && \
    mkdir -p /etc/otel && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

The runtime stage copies the virtual environment and application code from the builder stage, preserving the installed dependencies while eliminating build-time tools and intermediate files. It also copies OpenTelemetry configuration files for different monitoring backends:

```dockerfile
COPY --from=builder /app .
COPY otel/otel-collector-config-file.yaml /etc/otel/config-file.yaml
COPY otel/otel-collector-config-clickhouse.yaml /etc/otel/config-clickhouse.yaml
COPY otel/otel-collector-config-signoz.yaml /etc/otel/config-signoz.yaml
```

The Docker image exposes multiple ports for different services:
- Port 8283: Main Letta server API
- Port 5432: PostgreSQL database
- Ports 4317 and 4318: OpenTelemetry Collector endpoints for metrics and traces

The entrypoint is set to the standard PostgreSQL entrypoint script, with the command configured to run the Letta server startup script:

```dockerfile
EXPOSE 8283 5432 4317 4318
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
CMD ["./letta/server/startup.sh"]
```

This multi-stage build process results in a production-ready image that contains only the necessary components for running Letta, minimizing the attack surface and reducing image size.

**Section sources**
- [Dockerfile](file://Dockerfile#L1-L89)

## Production vs Development Deployment

Letta provides distinct Docker Compose configurations for production (`compose.yaml`) and development (`dev-compose.yaml`) environments, each optimized for its specific use case.

### Production Deployment Configuration
The production configuration in `compose.yaml` defines a three-service architecture with PostgreSQL database, Letta server, and NGINX reverse proxy:

```yaml
services:
  letta_db:
    image: ankane/pgvector:v0.5.1
    networks:
      default:
        aliases:
          - pgvector_db
          - letta-db
    environment:
      - POSTGRES_USER=${LETTA_PG_USER:-letta}
      - POSTGRES_PASSWORD=${LETTA_PG_PASSWORD:-letta}
      - POSTGRES_DB=${LETTA_PG_DB:-letta}
    volumes:
      - ./.persist/pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "${LETTA_PG_PORT:-5432}:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U letta"]
      interval: 5s
      timeout: 5s
      retries: 5
```

The database service uses environment variable substitution with defaults, ensuring flexibility across different deployment environments. Data persistence is achieved through volume mounting, with the database data stored in the `.persist/pgdata` directory and initialization scripts in `init.sql`.

The Letta server service is configured to use a pre-built image from Docker Hub:

```yaml
  letta_server:
    image: letta/letta:latest
    hostname: letta-server
    depends_on:
      letta_db:
        condition: service_healthy
    ports:
      - "8083:8083"
      - "8283:8283"
    env_file:
      - .env
    environment:
      - LETTA_PG_URI=${LETTA_PG_URI:-postgresql://${LETTA_PG_USER:-letta}:${LETTA_PG_PASSWORD:-letta}@${LETTA_DB_HOST:-letta-db}:${LETTA_PG_PORT:-5432}/${LETTA_PG_DB:-letta}}
      - LETTA_DEBUG=True
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      # ... other LLM provider API keys
```

Key production configuration features include:
- Health check dependency on the database service
- Environment file for sensitive configuration
- Dynamic PostgreSQL URI construction with defaults
- Support for multiple LLM providers through environment variables

The NGINX reverse proxy service provides HTTP routing and load balancing:

```yaml
  letta_nginx:
    hostname: letta-nginx
    image: nginx:stable-alpine3.17-slim
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "80:80"
```

### Development Deployment Configuration
The development configuration in `dev-compose.yaml` is optimized for local development with faster iteration cycles:

```yaml
services:
  letta_db:
    image: ankane/pgvector:v0.5.1
    environment:
      - POSTGRES_USER=${LETTA_PG_USER:-letta}
      - POSTGRES_PASSWORD=${LETTA_PG_PASSWORD:-letta}
      - POSTGRES_DB=${LETTA_PG_DB:-letta}
    volumes:
      - ./.persist/pgdata-test:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
```

The key difference in the development database configuration is the use of a separate data directory (`.persist/pgdata-test`) to isolate test data from production data.

The Letta server service in development builds the image locally rather than using a pre-built image:

```yaml
  letta_server:
    image: letta/letta:latest
    hostname: letta
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    depends_on:
      - letta_db
    ports:
      - "8083:8083"
      - "8283:8283"
    environment:
      # ... environment variables
```

This allows developers to make code changes and rebuild the image without pushing to a registry. The `target: runtime` parameter ensures that only the runtime stage of the Dockerfile is built, speeding up the build process.

An additional development configuration file, `development.compose.yml`, provides an even more developer-friendly setup with live code reloading:

```yaml
services:
  letta_server:
    image: letta_server
    build:
      context: .
      dockerfile: Dockerfile
      target: development
      args:
        - MEMGPT_ENVIRONMENT=DEVELOPMENT
    volumes:
      - ./letta:/letta
      - ~/.letta/credentials:/root/.letta/credentials
      - ./configs/server_config.yaml:/root/.letta/config
    environment:
      - WATCHFILES_FORCE_POLLING=true
```

This configuration mounts the local `letta` directory into the container, enabling live code reloading when files are modified. It also mounts configuration and credential files from the host system, making it easier to work with local development settings.

The key differences between production and development deployments are:

| Aspect | Production | Development |
|------|------------|-------------|
| Image Source | Pre-built from registry | Built locally |
| Data Persistence | Persistent volumes | Separate test data directory |
| Code Updates | Image rebuild and redeploy | Live code reloading |
| Debugging | Limited | Enhanced with debug tools |
| Performance | Optimized | Development-focused |
| Security | Hardened | More permissive |

**Section sources**
- [compose.yaml](file://compose.yaml#L1-L66)
- [dev-compose.yaml](file://dev-compose.yaml#L1-L49)
- [development.compose.yml](file://development.compose.yml#L1-L30)

## Image Building and Publishing Process

The Letta image building and publishing process is automated through the `pack_docker.sh` script, which handles versioning, multi-platform builds, and image pushing to the container registry.

### Build Script Analysis
The `pack_docker.sh` script orchestrates the entire image building and publishing workflow:

```bash
export MEMGPT_VERSION=$(letta version)
docker buildx build --platform=linux/amd64,linux/arm64,linux/x86_64 --build-arg MEMGPT_ENVIRONMENT=RELEASE -t letta/letta-server:${MEMGPT_VERSION} .
docker push letta/letta-server:${MEMGPT_VERSION}
```

The script performs three main operations:

1. **Version Retrieval**: The script first retrieves the current version of Letta using the `letta version` command and exports it as an environment variable. This ensures that the Docker image is tagged with the correct version number that matches the codebase.

2. **Multi-Platform Image Building**: The script uses `docker buildx` to build the Docker image for multiple platforms simultaneously:
   - `linux/amd64`: Standard x86-64 architecture
   - `linux/arm64`: ARM 64-bit architecture (used in Apple Silicon Macs and some cloud instances)
   - `linux/x86_64`: Alternative notation for x86-64 architecture

   This multi-platform support ensures that the Letta image can run on various hardware architectures without requiring separate build processes.

3. **Image Tagging and Pushing**: The built image is tagged with the retrieved version number and pushed to the Docker Hub repository `letta/letta-server`. This makes the image available for deployment in various environments.

### Build Process Details
The build process uses several important Docker features:

- **Build Arguments**: The script passes `MEMGPT_ENVIRONMENT=RELEASE` as a build argument, which can be used in the Dockerfile to customize the build process for release builds. This allows for different configurations between development and production builds.

- **Buildx Builder**: The use of `docker buildx` enables advanced Docker build features, including multi-platform builds and improved caching. This requires the Docker Buildx plugin, which is included in modern Docker installations.

- **Context Directory**: The build context is set to the current directory (`.`), which includes all necessary files for the build process, including the Dockerfile, source code, and dependency files.

The build process follows best practices for container image creation:
- It creates a single image with a clear version tag
- It supports multiple architectures for broad compatibility
- It automates the entire process from version retrieval to image publishing
- It uses semantic versioning for image tags

This automated process ensures consistency across builds and reduces the potential for human error in the release process. It also enables continuous integration and continuous deployment (CI/CD) workflows, where new versions can be automatically built and published when changes are merged to the main branch.

**Section sources**
- [scripts/pack_docker.sh](file://scripts/pack_docker.sh#L1-L4)

## Deployment Configuration and Environment Setup

### Environment Variable Management
Letta uses environment variables extensively for configuration, allowing for flexible deployment across different environments. The main configuration parameters include:

- **Database Configuration**: 
  - `LETTA_PG_USER`, `LETTA_PG_PASSWORD`, `LETTA_PG_DB`: PostgreSQL credentials and database name
  - `LETTA_PG_URI`: Complete PostgreSQL connection URI
  - `LETTA_DB_HOST`: Database host address

- **LLM Provider Configuration**:
  - `OPENAI_API_KEY`: OpenAI API key
  - `GROQ_API_KEY`: Groq API key
  - `ANTHROPIC_API_KEY`: Anthropic API key
  - `OLLAMA_BASE_URL`: Ollama server URL
  - `AZURE_API_KEY`, `AZURE_BASE_URL`, `AZURE_API_VERSION`: Azure AI services credentials
  - `GEMINI_API_KEY`: Google Gemini API key
  - `VLLM_API_BASE`: vLLM server URL

- **Observability Configuration**:
  - `LETTA_OTEL_EXPORTER_OTLP_ENDPOINT`: OpenTelemetry endpoint
  - `CLICKHOUSE_ENDPOINT`, `CLICKHOUSE_DATABASE`, `CLICKHOUSE_USERNAME`, `CLICKHOUSE_PASSWORD`: ClickHouse database credentials
  - `SIGNOZ_ENDPOINT`, `SIGNOZ_INGESTION_KEY`: Signoz monitoring credentials

These environment variables can be provided through the `.env` file or directly in the Docker Compose configuration, with default values provided for optional parameters.

### Secret Management
Letta implements secure secret management through multiple mechanisms:

1. **Docker Secrets**: The `init.sql` file demonstrates the use of Docker secrets for database credentials:
   ```sql
   \set db_user `([ -r /var/run/secrets/letta-user ] && cat /var/run/secrets/letta-user) || echo "${POSTGRES_USER:-letta}"`
   \set db_password `([ -r /var/run/secrets/letta-password ] && cat /var/run/secrets/letta-password) || echo "${POSTGRES_PASSWORD:-letta}"`
   \set db_name `([ -r /var/run/secrets/letta-db ] && cat /var/run/secrets/letta-db) || echo "${POSTGRES_DB:-letta}"`
   ```
   This approach allows secrets to be mounted as files in the container, providing an additional layer of security compared to environment variables.

2. **Environment Variable Fallbacks**: The configuration uses default values for environment variables, allowing for flexible deployment while maintaining security:
   ```yaml
   - POSTGRES_USER=${LETTA_PG_USER:-letta}
   ```
   This syntax uses the value of `LETTA_PG_USER` if defined, falling back to `letta` if not specified.

3. **Credential Files**: The development configuration mounts credential files from the host system:
   ```yaml
   - ~/.letta/credentials:/root/.letta/credentials
   ```
   This allows developers to use their local credentials without exposing them in version control.

### Container Networking
Letta's container networking is designed to facilitate communication between services while maintaining security:

1. **Internal Network**: Docker Compose creates a default network for the services, allowing them to communicate using service names as hostnames. For example, the Letta server can connect to the database using `letta-db` as the hostname.

2. **Port Exposure**: The services expose specific ports for external access:
   - Letta server: Ports 8083 and 8283
   - PostgreSQL: Port 5432
   - NGINX: Port 80

3. **Reverse Proxy**: The NGINX service acts as a reverse proxy, routing incoming HTTP requests to the Letta server:
   ```nginx
   location / {
       proxy_set_header Host $host;
       proxy_set_header X-Forwarded-For $remote_addr;
       proxy_set_header X-Forwarded-Proto $scheme;
       resolver 127.0.0.11; # docker dns
       proxy_pass $api_target;
   }
   ```
   This configuration preserves client information and enables proper routing within the Docker network.

### Initialization Process
The deployment initialization process is orchestrated through several components:

1. **Database Initialization**: The `init.sql` file is automatically executed when the PostgreSQL container starts, setting up the database schema and extensions:
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector WITH SCHEMA :"db_name";
   DROP SCHEMA IF EXISTS public CASCADE;
   ```
   This ensures that the vector database extension is available and that the public schema is removed for security.

2. **Server Startup**: The `startup.sh` script manages the server initialization process:
   - Waits for the PostgreSQL database to be ready
   - Performs database migrations using Alembic
   - Starts the OpenTelemetry Collector based on configuration
   - Launches the Letta server

3. **Health Checks**: The production configuration includes health checks for the database service:
   ```yaml
   healthcheck:
     test: ["CMD-SHELL", "pg_isready -U letta"]
     interval: 5s
     timeout: 5s
     retries: 5
   ```
   This ensures that the Letta server only starts after the database is fully operational.

### Step-by-Step Deployment Instructions
To deploy Letta in various environments, follow these steps:

1. **Prepare Environment Variables**: Create a `.env` file with the required configuration:
   ```env
   LETTA_PG_USER=letta
   LETTA_PG_PASSWORD=securepassword
   LETTA_PG_DB=letta
   OPENAI_API_KEY=your_openai_key
   # ... other required variables
   ```

2. **Choose Deployment Configuration**: Select the appropriate Docker Compose file:
   - `compose.yaml` for production
   - `dev-compose.yaml` for development
   - `development.compose.yml` for development with live reloading

3. **Start the Services**: Use Docker Compose to start the services:
   ```bash
   docker-compose -f compose.yaml up -d
   ```

4. **Verify Deployment**: Check the service status and logs:
   ```bash
   docker-compose -f compose.yaml ps
   docker-compose -f compose.yaml logs letta_server
   ```

5. **Access the Service**: The Letta API will be available at `http://localhost:8283` (or through the NGINX proxy at `http://localhost`).

This deployment process can be adapted for different environments by modifying the environment variables and configuration files accordingly.

**Section sources**
- [init.sql](file://init.sql#L1-L37)
- [nginx.conf](file://nginx.conf#L1-L29)
- [letta/server/startup.sh](file://letta/server/startup.sh#L1-L82)

## Container Security and Best Practices

### Security Configuration
Letta's container deployment incorporates several security best practices:

1. **Minimal Base Images**: The use of specialized base images like `ankane/pgvector:v0.5.1` and `nginx:stable-alpine3.17-slim` ensures that containers contain only the necessary components, reducing the attack surface.

2. **Separation of Build and Runtime**: The multi-stage Docker build process separates build-time dependencies from runtime dependencies, resulting in a leaner production image that doesn't include build tools or intermediate files.

3. **Non-Root User**: While not explicitly shown in the configuration, best practices suggest running containers as non-root users when possible to limit potential damage from security vulnerabilities.

4. **Secret Management**: The use of Docker secrets and environment variable fallbacks provides multiple layers of protection for sensitive information.

### Resource Allocation
Proper resource allocation is critical for stable Letta operation:

1. **Memory Configuration**: Letta's memory-intensive operations, particularly with large language models, require sufficient memory allocation. In Docker, this can be configured using the `mem_limit` parameter in the compose file.

2. **CPU Allocation**: The container should be allocated sufficient CPU resources, especially when handling multiple concurrent requests or complex AI operations.

3. **Storage Optimization**: The use of named volumes for database storage ensures data persistence and optimal performance:
   ```yaml
   volumes:
     - ./.persist/pgdata:/var/lib/postgresql/data
   ```

### Health Checks and Monitoring
Letta includes comprehensive health checks and monitoring capabilities:

1. **Database Health Check**: The production configuration includes a health check for the PostgreSQL service:
   ```yaml
   healthcheck:
     test: ["CMD-SHELL", "pg_isready -U letta"]
     interval: 5s
     timeout: 5s
     retries: 5
   ```
   This ensures that the database is fully operational before the application attempts to connect.

2. **OpenTelemetry Integration**: The deployment includes the OpenTelemetry Collector for comprehensive observability:
   ```dockerfile
   COPY otel/otel-collector-config-file.yaml /etc/otel/config-file.yaml
   COPY otel/otel-collector-config-clickhouse.yaml /etc/otel/config-clickhouse.yaml
   COPY otel/otel-collector-config-signoz.yaml /etc/otel/config-signoz.yaml
   ```
   This enables metrics, logs, and traces collection for monitoring and debugging.

3. **Startup Script Validation**: The `startup.sh` script includes validation steps to ensure proper initialization:
   ```bash
   wait_for_postgres() {
       until pg_isready -U "${POSTGRES_USER:-letta}" -h localhost; do
           echo "Waiting for PostgreSQL to be ready..."
           sleep 2
       done
   }
   ```
   This prevents the application from starting with an unavailable database.

### Network Security
The deployment configuration includes several network security measures:

1. **Internal Networking**: Services communicate over an internal Docker network, reducing exposure to external threats.

2. **Reverse Proxy**: The NGINX reverse proxy provides an additional layer of security by handling incoming requests and forwarding them to the application server.

3. **Port Minimization**: Only necessary ports are exposed to the host system, reducing the attack surface.

### Best Practices Summary
When deploying Letta in containerized environments, follow these best practices:

1. **Regular Updates**: Keep the base images and dependencies up to date to benefit from security patches and improvements.

2. **Environment Separation**: Use separate environments for development, testing, and production, with appropriate configuration for each.

3. **Backup Strategy**: Implement a regular backup strategy for the database volume to prevent data loss.

4. **Monitoring and Alerting**: Set up monitoring for key metrics such as CPU usage, memory consumption, and request latency.

5. **Security Scanning**: Regularly scan container images for vulnerabilities using tools like Docker Scout or Trivy.

6. **Resource Limits**: Set appropriate resource limits to prevent a single container from consuming excessive resources.

7. **Logging**: Ensure comprehensive logging is enabled and logs are regularly reviewed for security incidents.

8. **Access Control**: Restrict access to the deployment environment and management interfaces.

Following these best practices will help ensure a secure, reliable, and performant Letta deployment.

**Section sources**
- [Dockerfile](file://Dockerfile#L48-L88)
- [compose.yaml](file://compose.yaml#L1-L66)
- [letta/server/startup.sh](file://letta/server/startup.sh#L1-L82)

## Common Deployment Issues and Solutions

### Database Connection Problems
Database connection issues are among the most common deployment problems. Symptoms include startup failures, timeouts, and authentication errors.

**Solutions:**
1. **Verify Environment Variables**: Ensure that database credentials and connection parameters are correctly set in the environment variables or `.env` file.

2. **Check Database Health**: Use the health check to verify that the database is operational:
   ```bash
   docker-compose -f compose.yaml exec letta_db pg_isready -U letta
   ```

3. **Verify Network Connectivity**: Ensure that the Letta server can reach the database container:
   ```bash
   docker-compose -f compose.yaml exec letta_server ping letta-db
   ```

4. **Check Volume Permissions**: Ensure that the database volume has the correct permissions:
   ```bash
   docker-compose -f compose.yaml exec letta_db ls -la /var/lib/postgresql/data
   ```

5. **Review Logs**: Check the database container logs for initialization errors:
   ```bash
   docker-compose -f compose.yaml logs letta_db
   ```

### Port Conflicts
Port conflicts occur when the required ports are already in use by other services.

**Solutions:**
1. **Check Port Availability**: Verify that ports 80, 8083, 8283, and 5432 are not in use:
   ```bash
   netstat -tuln | grep -E '80|8083|8283|5432'
   ```

2. **Modify Port Mappings**: Change the port mappings in the Docker Compose file to use alternative ports:
   ```yaml
   ports:
     - "8084:8083"
     - "8284:8283"
   ```

3. **Stop Conflicting Services**: Stop services that are using the required ports:
   ```bash
   sudo systemctl stop apache2  # if using port 80
   ```

4. **Use Different Host Ports**: Keep the container ports the same but map them to different host ports:
   ```yaml
   ports:
     - "8083:8083"
     - "8283:8283"
   ```

### Configuration Issues
Configuration problems can lead to unexpected behavior or startup failures.

**Solutions:**
1. **Verify Environment Variables**: Ensure all required environment variables are set and have valid values.

2. **Check File Paths**: Verify that mounted volumes and file paths are correct:
   ```bash
   docker-compose -f compose.yaml exec letta_server ls -la /root/.letta/
   ```

3. **Validate Configuration Files**: Ensure that configuration files like `nginx.conf` are syntactically correct.

4. **Review Startup Script**: Check the `startup.sh` script for any errors in the initialization process.

### Performance Issues
Performance problems can manifest as slow response times or high resource utilization.

**Solutions:**
1. **Monitor Resource Usage**: Use Docker commands to monitor CPU and memory usage:
   ```bash
   docker stats
   ```

2. **Optimize Database**: Ensure the database has appropriate indexes and is properly tuned.

3. **Adjust Resource Limits**: Increase memory and CPU allocation if resources are constrained.

4. **Enable Caching**: Ensure that any available caching mechanisms are properly configured.

### Debugging and Troubleshooting
When encountering deployment issues, follow this systematic approach:

1. **Check Service Status**: Verify the status of all services:
   ```bash
   docker-compose -f compose.yaml ps
   ```

2. **Review Logs**: Examine the logs for error messages:
   ```bash
   docker-compose -f compose.yaml logs --tail=50 letta_server
   ```

3. **Test Connectivity**: Verify network connectivity between services:
   ```bash
   docker-compose -f compose.yaml exec letta_server curl -v http://letta-db:5432
   ```

4. **Validate Configuration**: Check that all configuration files are correctly formatted and contain valid values.

5. **Restart Services**: Sometimes a simple restart resolves transient issues:
   ```bash
   docker-compose -f compose.yaml restart
   ```

6. **Check Disk Space**: Ensure sufficient disk space is available for the containers:
   ```bash
   df -h
   ```

By following these troubleshooting steps, most deployment issues can be identified and resolved efficiently.

**Section sources**
- [compose.yaml](file://compose.yaml#L1-L66)
- [dev-compose.yaml](file://dev-compose.yaml#L1-L49)
- [letta/server/startup.sh](file://letta/server/startup.sh#L1-L82)

## Conclusion
The containerized deployment of Letta using Docker and Docker Compose provides a robust, flexible, and scalable solution for deploying AI agents with advanced memory capabilities. The multi-stage Docker build process ensures optimized images for both development and production environments, while the comprehensive Docker Compose configurations enable easy deployment and management of the entire application stack.

Key strengths of the deployment architecture include:
- **Multi-stage builds** that separate build-time and runtime dependencies
- **Flexible configuration** through environment variables and compose files
- **Comprehensive observability** with OpenTelemetry integration
- **Secure secret management** using Docker secrets and environment variable fallbacks
- **Robust initialization** with health checks and startup validation

The deployment system supports both production and development workflows, with optimized configurations for each environment. The automated image building and publishing process ensures consistent releases across different platforms and architectures.

When deploying Letta, it's important to follow security best practices, properly configure resources, and implement monitoring and alerting. The provided troubleshooting guidance helps quickly resolve common issues related to database connectivity, port conflicts, and configuration problems.

Overall, the containerized deployment approach enables organizations to leverage Letta's advanced AI capabilities in a reliable and scalable manner, whether for local development, cloud deployment, or on-premise installations.

**Section sources**
- [Dockerfile](file://Dockerfile#L1-L89)
- [compose.yaml](file://compose.yaml#L1-L66)
- [dev-compose.yaml](file://dev-compose.yaml#L1-L49)
- [scripts/pack_docker.sh](file://scripts/pack_docker.sh#L1-L4)