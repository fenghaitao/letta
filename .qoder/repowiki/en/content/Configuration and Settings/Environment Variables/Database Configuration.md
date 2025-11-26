# Database Configuration

<cite>
**Referenced Files in This Document**
- [letta/settings.py](file://letta/settings.py)
- [letta/database_utils.py](file://letta/database_utils.py)
- [letta/server/db.py](file://letta/server/db.py)
- [letta/orm/sqlalchemy_base.py](file://letta/orm/sqlalchemy_base.py)
- [letta/orm/sqlite_functions.py](file://letta/orm/sqlite_functions.py)
- [letta/orm/errors.py](file://letta/orm/errors.py)
- [letta/helpers/crypto_utils.py](file://letta/helpers/crypto_utils.py)
- [letta/schemas/enums.py](file://letta/schemas/enums.py)
- [alembic/env.py](file://alembic/env.py)
- [alembic/versions/8149a781ac1b_backfill_encrypted_columns_for_.py](file://alembic/versions/8149a781ac1b_backfill_encrypted_columns_for_.py)
- [letta/otel/db_pool_monitoring.py](file://letta/otel/db_pool_monitoring.py)
- [letta/otel/metric_registry.py](file://letta/otel/metric_registry.py)
- [init.sql](file://init.sql)
- [letta/server/startup.sh](file://letta/server/startup.sh)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Database Configuration Overview](#database-configuration-overview)
3. [Environment Variables](#environment-variables)
4. [Database Engine Selection](#database-engine-selection)
5. [Connection Strings and URI Formats](#connection-strings-and-uri-formats)
6. [Connection Pool Configuration](#connection-pool-configuration)
7. [Security and Encryption](#security-and-encryption)
8. [Database Migration and Schema Management](#database-migration-and-schema-management)
9. [Monitoring and Metrics](#monitoring-and-metrics)
10. [Troubleshooting Guide](#troubleshooting-guide)
11. [Best Practices](#best-practices)

## Introduction

Letta provides flexible database configuration options to support both development and production environments. The system supports two primary database engines: PostgreSQL and SQLite, with PostgreSQL being the recommended choice for production deployments. This comprehensive guide covers all aspects of database configuration, from basic environment variables to advanced connection pooling and security considerations.

## Database Configuration Overview

Letta's database configuration is managed through a centralized settings system that supports multiple configuration methods and provides automatic fallback mechanisms. The system automatically detects the database type and establishes connections based on available configuration options.

```mermaid
flowchart TD
A["Database Configuration"] --> B["Check pg_uri"]
B --> |Present| C["Use Full URI"]
B --> |Missing| D["Check Individual Parameters"]
D --> |All Present| E["Build URI from Parameters"]
D --> |Missing| F["Use Default SQLite"]
C --> G["Determine Database Engine"]
E --> G
F --> H["SQLite Engine"]
G --> I{"Database Choice"}
I --> |Has pg_uri| J["PostgreSQL Engine"]
I --> |No pg_uri| H
J --> K["Establish Connection"]
H --> K
K --> L["Configure Pooling"]
L --> M["Ready for Operations"]
```

**Diagram sources**
- [letta/settings.py](file://letta/settings.py#L361-L384)
- [letta/server/db.py](file://letta/server/db.py#L16-L46)

**Section sources**
- [letta/settings.py](file://letta/settings.py#L361-L384)
- [letta/server/db.py](file://letta/server/db.py#L16-L46)

## Environment Variables

Letta uses a comprehensive set of environment variables to configure database connections. These variables provide flexibility in configuring both individual connection parameters and complete connection URIs.

### Core Database Variables

| Variable | Description | Default Value | Example |
|----------|-------------|---------------|---------|
| `LETTA_PG_URI` | Complete PostgreSQL connection URI | None | `postgresql://user:pass@host:5432/db` |
| `LETTA_PG_DB` | Database name | None | `letta` |
| `LETTA_PG_USER` | Database username | None | `letta` |
| `LETTA_PG_PASSWORD` | Database password | None | `password` |
| `LETTA_PG_HOST` | Database host | None | `localhost` |
| `LETTA_PG_PORT` | Database port | None | `5432` |

### Connection Pool Variables

| Variable | Description | Default Value | Purpose |
|----------|-------------|---------------|---------|
| `LETTA_PG_POOL_SIZE` | Initial pool size | 25 | Maximum concurrent connections |
| `LETTA_PG_MAX_OVERFLOW` | Maximum overflow connections | 10 | Connections beyond pool size |
| `LETTA_PG_POOL_TIMEOUT` | Connection timeout (seconds) | 30 | Time to wait for connection |
| `LETTA_PG_POOL_RECYCLE` | Connection recycling time (seconds) | 1800 | When to recycle connections |
| `LETTA_DISABLE_SQLALCHEMY_POOLING` | Disable SQLAlchemy pooling | True | Use NullPool for testing |

### Advanced Configuration Variables

| Variable | Description | Default Value | Impact |
|----------|-------------|---------------|--------|
| `LETTA_PG_ECHO` | Enable SQL logging | False | Debug database operations |
| `LETTA_POOL_PRE_PING` | Enable pre-ping checks | True | Detect dead connections |
| `LETTA_POOL_USE_LIFO` | Use LIFO queue policy | True | Connection reuse strategy |

**Section sources**
- [letta/settings.py](file://letta/settings.py#L249-L263)

## Database Engine Selection

Letta automatically selects the database engine based on configuration availability. The selection process follows a hierarchical approach with clear fallback mechanisms.

```mermaid
classDiagram
class DatabaseChoice {
<<enumeration>>
POSTGRES
SQLITE
}
class Settings {
+letta_pg_uri : str
+pg_db : str
+pg_user : str
+pg_password : str
+pg_host : str
+pg_port : int
+database_engine : DatabaseChoice
+letta_pg_uri_no_default : str
}
Settings --> DatabaseChoice : determines
DatabaseChoice --> POSTGRES : when pg_uri present
DatabaseChoice --> SQLITE : when no pg_uri
```

**Diagram sources**
- [letta/settings.py](file://letta/settings.py#L225-L228)
- [letta/settings.py](file://letta/settings.py#L381-L384)

### Engine Selection Logic

The database engine determination follows this logic:

1. **PostgreSQL Engine**: Selected when `letta_pg_uri_no_default` returns a valid URI
2. **SQLite Engine**: Selected when no PostgreSQL URI is configured

### Database Engine Properties

The system provides several properties to access database configuration:

- `letta_pg_uri`: Returns complete URI with fallback to default
- `letta_pg_uri_no_default`: Returns URI or None (used for engine detection)
- `database_engine`: Returns DatabaseChoice enumeration

**Section sources**
- [letta/settings.py](file://letta/settings.py#L361-L384)

## Connection Strings and URI Formats

Letta supports both full connection URIs and individual connection parameters. The system automatically converts between formats as needed.

### Full URI Format

The complete PostgreSQL URI format includes all connection parameters:

```
postgresql+driver://username:password@host:port/database?query_options
```

Example:
```
postgresql+pg8000://letta:password@localhost:5432/letta?sslmode=require&application_name=letta
```

### Individual Parameter Format

When using individual parameters, Letta constructs the URI internally:

```python
# Constructed URI format
f"postgresql+pg8000://{self.pg_user}:{self.pg_password}@{self.pg_host}:{self.pg_port}/{self.pg_db}"
```

### URI Conversion Utilities

Letta provides utilities for converting between different driver formats:

```mermaid
flowchart LR
A["Original URI"] --> B["Parse URI"]
B --> C["Extract Components"]
C --> D["Convert Driver"]
D --> E["Build New URI"]
E --> F["asyncpg URI"]
E --> G["pg8000 URI"]
F --> H["Async Operations"]
G --> I["Sync Operations"]
```

**Diagram sources**
- [letta/database_utils.py](file://letta/database_utils.py#L97-L161)

### Default Connection Fallback

When no configuration is provided, Letta falls back to a default SQLite database:

```
postgresql+pg8000://letta:letta@localhost:5432/letta
```

**Section sources**
- [letta/settings.py](file://letta/settings.py#L361-L368)
- [letta/database_utils.py](file://letta/database_utils.py#L43-L94)

## Connection Pool Configuration

Letta uses SQLAlchemy's connection pooling with configurable parameters for optimal performance in different deployment scenarios.

### Pool Configuration Parameters

| Parameter | Purpose | Recommended Values | Notes |
|-----------|---------|-------------------|-------|
| `pool_size` | Initial pool size | 25 | Concurrent connections |
| `max_overflow` | Overflow limit | 10 | Additional connections |
| `pool_timeout` | Connection timeout | 30 | Seconds to wait |
| `pool_recycle` | Connection recycling | 1800 | Seconds before recycle |
| `pool_pre_ping` | Pre-ping checks | True | Detect dead connections |

### Pool Monitoring and Metrics

Letta provides comprehensive pool monitoring capabilities:

```mermaid
graph TB
A["Database Pool"] --> B["Connection Monitoring"]
B --> C["Metrics Collection"]
C --> D["Pool Statistics"]
C --> E["Event Tracking"]
C --> F["Error Reporting"]
D --> G["Checked Out Connections"]
D --> H["Available Connections"]
D --> I["Overflow Connections"]
E --> J["Checkout Events"]
E --> K["Checkin Events"]
E --> L["Invalidate Events"]
F --> M["Connection Errors"]
F --> N["Timeout Errors"]
```

**Diagram sources**
- [letta/otel/db_pool_monitoring.py](file://letta/otel/db_pool_monitoring.py#L69-L309)
- [letta/otel/metric_registry.py](file://letta/otel/metric_registry.py#L199-L275)

### Pool Configuration Examples

#### Production Configuration
```bash
LETTA_PG_POOL_SIZE=50
LETTA_PG_MAX_OVERFLOW=20
LETTA_PG_POOL_TIMEOUT=60
LETTA_PG_POOL_RECYCLE=3600
LETTA_POOL_PRE_PING=true
```

#### Development Configuration
```bash
LETTA_PG_POOL_SIZE=5
LETTA_PG_MAX_OVERFLOW=5
LETTA_PG_POOL_TIMEOUT=30
LETTA_PG_POOL_RECYCLE=1800
LETTA_DISABLE_SQLALCHEMY_POOLING=false
```

**Section sources**
- [letta/server/db.py](file://letta/server/db.py#L25-L46)
- [letta/otel/db_pool_monitoring.py](file://letta/otel/db_pool_monitoring.py#L273-L295)

## Security and Encryption

Letta implements robust security measures for database credential protection and data encryption.

### Encryption Key Configuration

The `LETTA_ENCRYPTION_KEY` environment variable enables secure storage of sensitive database credentials:

```bash
LETTA_ENCRYPTION_KEY=your-secure-encryption-key-here
```

### Credential Protection Mechanisms

```mermaid
sequenceDiagram
participant App as Application
participant Crypto as Crypto Utils
participant DB as Database
App->>Crypto : Encrypt Credentials
Crypto->>Crypto : Generate Salt & IV
Crypto->>Crypto : AES-256-GCM Encryption
Crypto-->>App : Encrypted Credentials
App->>DB : Connect with Encrypted Credentials
DB->>Crypto : Decrypt Credentials
Crypto-->>DB : Plain Text Credentials
DB-->>App : Database Connection
Note over App,DB : All credentials stored encrypted
```

**Diagram sources**
- [letta/helpers/crypto_utils.py](file://letta/helpers/crypto_utils.py#L40-L120)

### Supported Encryption Features

1. **AES-256-GCM Encryption**: Strong encryption with authentication
2. **Salt Generation**: Random salts for each encryption
3. **Automatic Migration**: Existing credentials can be encrypted
4. **Dual Read Support**: Backward compatibility with plaintext credentials

### Security Best Practices

- **Environment Variables**: Store encryption keys in environment variables
- **Secret Management**: Use Kubernetes secrets or similar for production
- **Key Rotation**: Regularly rotate encryption keys
- **Access Control**: Limit access to encryption keys

**Section sources**
- [letta/helpers/crypto_utils.py](file://letta/helpers/crypto_utils.py#L40-L120)
- [alembic/versions/8149a781ac1b_backfill_encrypted_columns_for_.py](file://alembic/versions/8149a781ac1b_backfill_encrypted_columns_for_.py#L27-L32)

## Database Migration and Schema Management

Letta uses Alembic for database schema migrations with automatic engine detection.

### Migration Configuration

The migration system automatically selects the appropriate database engine:

```mermaid
flowchart TD
A["Alembic Migration"] --> B{"Database Engine"}
B --> |PostgreSQL| C["Use asyncpg URI"]
B --> |SQLite| D["Use SQLite URI"]
C --> E["Apply Migrations"]
D --> F["Apply SQLite Migrations"]
E --> G["Schema Updated"]
F --> G
```

**Diagram sources**
- [alembic/env.py](file://alembic/env.py#L17-L27)

### Migration Process

1. **Engine Detection**: Alembic detects the database type
2. **URI Conversion**: Converts URI format for Alembic compatibility
3. **Migration Execution**: Applies schema changes
4. **Version Tracking**: Maintains migration history

### Migration Examples

#### PostgreSQL Migration
```bash
# Automatic conversion from asyncpg to pg8000
export LETTA_PG_URI="postgresql://user:pass@host:5432/db"
alembic upgrade head
```

#### SQLite Migration
```bash
# Uses built-in SQLite
export LETTA_PG_URI=""  # Empty to trigger SQLite
alembic upgrade head
```

**Section sources**
- [alembic/env.py](file://alembic/env.py#L17-L27)

## Monitoring and Metrics

Letta provides comprehensive database monitoring and metrics collection for operational insights.

### Available Metrics

| Metric Category | Metrics | Purpose |
|----------------|---------|---------|
| Connection Pool | Checked out, available, overflow | Pool utilization |
| Connection Events | Checkout, checkin, invalidate | Connection lifecycle |
| Connection Duration | Histogram of connection usage | Performance analysis |
| Connection Errors | Error counts by type | Troubleshooting |

### Monitoring Configuration

```bash
LETTA_ENABLE_DB_POOL_MONITORING=true
LETTA_DB_POOL_MONITORING_INTERVAL=30
```

### Metrics Collection Flow

```mermaid
graph LR
A["Database Operations"] --> B["Event Listeners"]
B --> C["Metric Registry"]
C --> D["Prometheus Metrics"]
C --> E["Log Output"]
C --> F["Tracing"]
D --> G["Monitoring Dashboard"]
E --> H["Operational Logs"]
F --> I["Distributed Tracing"]
```

**Diagram sources**
- [letta/otel/db_pool_monitoring.py](file://letta/otel/db_pool_monitoring.py#L69-L309)

**Section sources**
- [letta/otel/db_pool_monitoring.py](file://letta/otel/db_pool_monitoring.py#L69-L309)
- [letta/otel/metric_registry.py](file://letta/otel/metric_registry.py#L199-L275)

## Troubleshooting Guide

Common database configuration issues and their solutions.

### Authentication Failures

**Symptoms:**
- `psycopg2.OperationalError: FATAL: password authentication failed`
- Connection refused errors

**Solutions:**
1. Verify database credentials
2. Check database user permissions
3. Validate network connectivity
4. Ensure SSL requirements match

### Connection Timeouts

**Symptoms:**
- `TimeoutError: QueuePool limit of size X overflow Y reached`
- Slow database operations

**Solutions:**
1. Increase pool parameters:
   ```bash
   LETTA_PG_POOL_SIZE=100
   LETTA_PG_MAX_OVERFLOW=50
   LETTA_PG_POOL_TIMEOUT=60
   ```
2. Optimize database queries
3. Check network latency

### Pool Exhaustion

**Symptoms:**
- `QueuePool limit of size X overflow Y reached`
- Database connection errors

**Solutions:**
1. Monitor pool metrics
2. Adjust pool sizing based on load
3. Implement connection cleanup
4. Use connection pooling best practices

### Migration Issues

**Symptoms:**
- `alembic upgrade head` failures
- Schema version mismatches

**Solutions:**
1. Check database connectivity
2. Verify migration prerequisites
3. Review migration logs
4. Manual intervention may be required

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| `DatabaseTimeoutError` | Long-running queries | Increase timeout or optimize queries |
| `UniqueConstraintViolationError` | Duplicate entries | Check data uniqueness |
| `ForeignKeyConstraintViolationError` | Referential integrity | Fix relationship data |

**Section sources**
- [letta/orm/errors.py](file://letta/orm/errors.py#L17-L22)

## Best Practices

### Development Environment

```bash
# Development configuration
export LETTA_PG_URI=""
export LETTA_DEBUG=true
export LETTA_PG_ECHO=true
export LETTA_DISABLE_SQLALCHEMY_POOLING=true
```

### Production Environment

```bash
# Production configuration
export LETTA_PG_URI="postgresql://user:pass@host:5432/db"
export LETTA_PG_POOL_SIZE=50
export LETTA_PG_MAX_OVERFLOW=20
export LETTA_PG_POOL_TIMEOUT=60
export LETTA_POOL_PRE_PING=true
export LETTA_ENCRYPTION_KEY="your-production-key"
export LETTA_ENABLE_DB_POOL_MONITORING=true
```

### Security Recommendations

1. **Use Environment Variables**: Never hardcode credentials
2. **Encrypt Sensitive Data**: Enable `LETTA_ENCRYPTION_KEY`
3. **Network Security**: Use VPN or private networks
4. **Regular Updates**: Keep database software current
5. **Backup Strategy**: Implement regular backups

### Performance Tuning

1. **Monitor Metrics**: Track pool utilization and connection times
2. **Optimize Queries**: Use indexes and efficient query patterns
3. **Connection Pooling**: Tune pool parameters for workload
4. **Resource Limits**: Set appropriate limits for connections

### Deployment Considerations

1. **Database Selection**: Choose PostgreSQL for production
2. **High Availability**: Implement replication and failover
3. **Scaling**: Plan for horizontal scaling needs
4. **Monitoring**: Set up comprehensive monitoring