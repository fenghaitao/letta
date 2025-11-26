# Database Operations

<cite>
**Referenced Files in This Document**   
- [db/run_postgres.sh](file://db/run_postgres.sh)
- [init.sql](file://init.sql)
- [compose.yaml](file://compose.yaml)
- [dev-compose.yaml](file://dev-compose.yaml)
- [alembic.ini](file://alembic.ini)
- [alembic/env.py](file://alembic/env.py)
- [alembic/versions/9a505cc7eca9_create_a_baseline_migrations.py](file://alembic/versions/9a505cc7eca9_create_a_baseline_migrations.py)
- [alembic/versions/a113caac453e_add_identities_table.py](file://alembic/versions/a113caac453e_add_identities_table.py)
- [letta/server/db.py](file://letta/server/db.py)
- [letta/settings.py](file://letta/settings.py)
- [letta/database_utils.py](file://letta/database_utils.py)
- [letta/orm/sqlalchemy_base.py](file://letta/orm/sqlalchemy_base.py)
- [letta/orm/base.py](file://letta/orm/base.py)
</cite>

## Table of Contents
1. [Database Initialization](#database-initialization)
2. [Database Migration Strategy](#database-migration-strategy)
3. [Database Configuration](#database-configuration)
4. [Connection Pooling and Performance Tuning](#connection-pooling-and-performance-tuning)
5. [Database Maintenance](#database-maintenance)
6. [Backup and Recovery](#backup-and-recovery)
7. [High Availability and Replication](#high-availability-and-replication)
8. [Troubleshooting Common Issues](#troubleshooting-common-issues)

## Database Initialization

Letta's PostgreSQL database initialization process is designed to provide a robust, containerized environment with proper schema setup and data seeding. The initialization process leverages Docker containers and SQL scripts to ensure consistent database setup across different environments.

The database initialization is primarily handled through two key components: the `run_postgres.sh` script and the `init.sql` file. The `run_postgres.sh` script orchestrates the containerized PostgreSQL setup, building a custom Docker image and running a PostgreSQL container with specific configurations. This script creates a container named "letta-db-test" that exposes port 8888 on the host machine, mapping it to the standard PostgreSQL port 5432 within the container. The script sets the PostgreSQL password to "password" and mounts a Docker volume named "letta_db_test" to ensure data persistence across container restarts.

The `init.sql` file contains the SQL commands necessary to initialize the database schema and configure the PostgreSQL environment. This script implements a flexible approach to database configuration by first attempting to read database credentials from Docker secrets located at `/var/run/secrets/`, falling back to environment variables if secrets are not available, and finally using hardcoded defaults if neither secrets nor environment variables are present. The script creates a dedicated schema named after the database, sets it as the default search path, and enables the pgvector extension for vector operations. Notably, it removes the public schema to enhance security and ensure all database objects are created within the designated schema.

The initialization process is also integrated with Docker Compose through the `compose.yaml` and `dev-compose.yaml` files, which define services for both production and development environments. These configuration files mount the `init.sql` script to the container's initialization directory (`/docker-entrypoint-initdb.d/`), ensuring that the script executes automatically when the container starts for the first time. This approach guarantees that the database is properly configured with the correct schema, extensions, and security settings without requiring manual intervention.

**Section sources**
- [db/run_postgres.sh](file://db/run_postgres.sh)
- [init.sql](file://init.sql)
- [compose.yaml](file://compose.yaml)
- [dev-compose.yaml](file://dev-compose.yaml)

## Database Migration Strategy

Letta employs Alembic as its database migration framework to manage schema changes in a systematic and version-controlled manner. The migration strategy is designed to ensure database schema consistency across different environments and deployment stages while maintaining data integrity during upgrades.

The Alembic configuration is defined in the `alembic.ini` file, which specifies the location of migration scripts, logging settings, and other configuration parameters. The `env.py` file serves as the entry point for Alembic operations, configuring the migration environment by setting the database URI based on the application settings. This file dynamically determines whether to use PostgreSQL or SQLite and configures the appropriate database connection accordingly. The migration environment is set up to use the SQLAlchemy metadata from the application's ORM models, enabling Alembic to generate migration scripts that accurately reflect the application's data model.

Migration scripts are stored in the `alembic/versions/` directory and are executed in chronological order based on their revision IDs. Each migration script follows a standard structure with `upgrade()` and `downgrade()` functions that define the forward and backward operations for schema changes. For example, the baseline migration script `9a505cc7eca9_create_a_baseline_migrations.py` creates the initial database schema by defining tables for agents, messages, sources, and other core entities. Subsequent migrations, such as `a113caac453e_add_identities_table.py`, demonstrate the incremental evolution of the schema by adding new tables and modifying existing ones.

The migration process is designed to be database-agnostic, with conditional logic that skips migrations when running with SQLite. This allows developers to use SQLite for testing while maintaining PostgreSQL as the primary production database. The migration system also includes proper error handling and transaction management to ensure that schema changes are applied atomically and can be rolled back in case of failures. This comprehensive migration strategy enables Letta to evolve its data model over time while maintaining compatibility with existing data and supporting both forward and backward migrations for development and deployment flexibility.

**Section sources**
- [alembic.ini](file://alembic.ini)
- [alembic/env.py](file://alembic/env.py)
- [alembic/versions/9a505cc7eca9_create_a_baseline_migrations.py](file://alembic/versions/9a505cc7eca9_create_a_baseline_migrations.py)
- [alembic/versions/a113caac453e_add_identities_table.py](file://alembic/versions/a113caac453e_add_identities_table.py)

## Database Configuration

The database configuration in Letta is managed through a combination of Docker Compose files, environment variables, and application settings to provide a flexible and secure setup for both development and production environments. The primary configuration is defined in the `compose.yaml` file, which orchestrates the PostgreSQL database service alongside other application components.

The `compose.yaml` file configures the PostgreSQL database service using the `ankane/pgvector:v0.5.1` image, which includes the pgvector extension for vector operations essential to Letta's functionality. The service is configured with environment variables for the PostgreSQL user, password, and database name, which can be overridden using environment variables with default values. The configuration includes volume mounting for data persistence, mapping the host directory `./.persist/pgdata` to the container's PostgreSQL data directory `/var/lib/postgresql/data`. This ensures that database data persists across container restarts and deployments.

A critical aspect of the database configuration is the initialization process, where the `init.sql` script is mounted to the container's initialization directory `/docker-entrypoint-initdb.d/init.sql`. This ensures that the script executes automatically when the container starts for the first time, setting up the database schema, creating the custom schema, and configuring the pgvector extension. The service also includes a health check that verifies database availability using the `pg_isready` command, ensuring that dependent services like the Letta server only start once the database is fully operational.

The configuration also defines environment variables for the Letta server service, including the `LETTA_PG_URI` which specifies the connection string to the PostgreSQL database. This URI is constructed using environment variables with default values, providing flexibility for different deployment scenarios. The configuration supports environment-specific overrides through files like `dev-compose.yaml`, which provides alternative settings for development environments, including different volume mounts and build configurations. This layered configuration approach enables consistent database setup across different environments while allowing for necessary variations in development, testing, and production deployments.

**Section sources**
- [compose.yaml](file://compose.yaml)
- [dev-compose.yaml](file://dev-compose.yaml)
- [init.sql](file://init.sql)
- [letta/settings.py](file://letta/settings.py)

## Connection Pooling and Performance Tuning

Letta implements comprehensive connection pooling and performance tuning strategies to optimize database interactions and ensure efficient resource utilization. The connection pooling configuration is managed through application settings in the `settings.py` file, which defines key parameters for database connection management.

The database connection pool is configured with a maximum of 25 concurrent connections (`pg_pool_size`) and allows up to 10 overflow connections (`pg_max_overflow`) when the pool is exhausted. Connections are configured to timeout after 30 seconds (`pg_pool_timeout`) if they cannot be obtained, preventing indefinite blocking. The pool is set to recycle connections every 30 minutes (`pg_pool_recycle`) to prevent issues with stale or dead connections, and pre-pinging is enabled (`pool_pre_ping`) to verify connection health before use. These settings strike a balance between resource efficiency and connection availability, ensuring that the application can handle varying loads without exhausting database resources.

The actual database connection and session management is implemented in the `letta/server/db.py` file, which creates an asynchronous database engine using SQLAlchemy. The engine configuration varies based on the `disable_sqlalchemy_pooling` setting, allowing for different pooling strategies in development and production environments. When pooling is enabled, the engine uses the configured pool size, overflow, timeout, and recycle settings. The connection arguments include specific settings for asyncpg, such as disabling the statement cache and prepared statement cache to optimize performance for the application's access patterns.

Performance is further enhanced through the use of asynchronous database operations, which allow the application to handle multiple database requests concurrently without blocking. The database session factory is configured with appropriate settings for autocommit, autoflush, and expiration to optimize transaction management. Additionally, the application implements query optimization techniques such as proper indexing, efficient query construction, and batch operations for improved performance. The configuration also includes monitoring capabilities through the `enable_db_pool_monitoring` setting, which allows for collection of connection pool statistics to identify potential bottlenecks and optimize performance over time.

**Section sources**
- [letta/server/db.py](file://letta/server/db.py)
- [letta/settings.py](file://letta/settings.py)
- [letta/database_utils.py](file://letta/database_utils.py)

## Database Maintenance

Letta incorporates several database maintenance practices to ensure optimal performance, data integrity, and long-term stability of the PostgreSQL database. These maintenance operations are designed to address common database administration tasks while integrating seamlessly with the application's architecture and deployment model.

The ORM base classes in `letta/orm/base.py` and `letta/orm/sqlalchemy_base.py` implement common patterns for database maintenance, including automatic timestamp management and soft deletion. The `CommonSqlalchemyMetaMixins` class provides `created_at` and `updated_at` fields with automatic server-side defaults, ensuring consistent timestamp tracking across all database entities. The soft deletion pattern, implemented through the `is_deleted` field with a default value of FALSE, allows for logical deletion of records while preserving data for auditing and recovery purposes.

The application implements comprehensive error handling and constraint management to maintain data integrity. The `SqlalchemyBase` class includes methods for handling database errors such as unique constraint violations and foreign key constraint violations, converting low-level database exceptions into meaningful application-level exceptions. This approach helps prevent data corruption and provides clear error messages for troubleshooting. The base class also includes methods for batch operations, such as `batch_create_async`, which optimizes performance by reducing the number of round trips to the database for bulk operations.

Maintenance operations are further supported by the application's configuration and utility functions. The `database_utils.py` file provides utilities for parsing and converting database URIs, ensuring consistent connection handling across different components of the application. The ORM models include methods for efficient querying, pagination, and filtering, which help prevent performance issues related to large result sets. Additionally, the application implements proper transaction management and connection handling to prevent connection leaks and ensure data consistency during concurrent operations.

**Section sources**
- [letta/orm/sqlalchemy_base.py](file://letta/orm/sqlalchemy_base.py)
- [letta/orm/base.py](file://letta/orm/base.py)
- [letta/database_utils.py](file://letta/database_utils.py)

## Backup and Recovery

Letta's architecture supports robust backup and recovery procedures through its containerized deployment model and persistent storage configuration. While the codebase does not include explicit backup scripts, the design facilitates reliable backup and recovery operations through standard PostgreSQL mechanisms and Docker volume management.

The primary mechanism for data persistence and backup is the volume mounting configuration in the `compose.yaml` file, which maps the host directory `./.persist/pgdata` to the container's PostgreSQL data directory. This approach allows for straightforward backup operations by simply copying the contents of the mounted directory. Since PostgreSQL stores all database data in this directory, creating a backup involves making a consistent copy of the entire directory structure while ensuring database consistency.

For production deployments, administrators can implement regular backup procedures using standard PostgreSQL tools such as `pg_dump` and `pg_dumpall`. These tools can be executed within the running container using Docker exec commands, allowing for both full database dumps and incremental backups. The containerized environment ensures that backup operations can be automated through scripts that execute these commands at scheduled intervals, with backup files stored in designated backup locations.

Recovery procedures leverage the same volume mounting mechanism, allowing for restoration of database state by replacing the contents of the persistent volume with a previous backup. In the event of data corruption or accidental deletion, administrators can stop the PostgreSQL container, replace the data directory with a backup, and restart the container to restore the database to a previous state. The initialization process, which only executes on first container startup, ensures that existing data is preserved during container restarts, making the recovery process reliable and predictable.

**Section sources**
- [compose.yaml](file://compose.yaml)
- [dev-compose.yaml](file://dev-compose.yaml)

## High Availability and Replication

While the current configuration files primarily focus on single-instance database deployments, Letta's architecture supports high availability and replication configurations for production environments. The design principles and configuration options enable the implementation of robust high availability setups through external PostgreSQL configuration and deployment strategies.

The application's database configuration is designed to be flexible, allowing for connection to external PostgreSQL instances or clusters rather than being limited to the containerized database defined in the compose files. By modifying the `LETTA_PG_URI` environment variable, the application can connect to a PostgreSQL cluster with built-in replication and failover capabilities. This approach enables administrators to implement high availability using established PostgreSQL solutions such as Patroni, PgBouncer, or cloud-managed PostgreSQL services with built-in replication.

The connection pooling and retry mechanisms implemented in the application contribute to high availability by providing resilience to temporary database connectivity issues. The connection pool's pre-ping feature (`pool_pre_ping`) helps detect and replace dead connections, while the timeout settings prevent indefinite blocking during network issues. These features work in conjunction with external load balancers and failover mechanisms to provide a seamless experience during database failover events.

For production deployments requiring high availability, administrators can implement master-slave replication with automatic failover, using the application's configuration to direct read operations to replica instances and write operations to the primary instance. The application's asynchronous database operations and connection pooling are well-suited to this architecture, efficiently utilizing multiple database instances to distribute load and improve performance. Additionally, the use of environment variables for database configuration allows for dynamic reconfiguration of database endpoints in response to failover events or maintenance operations.

**Section sources**
- [compose.yaml](file://compose.yaml)
- [letta/settings.py](file://letta/settings.py)
- [letta/server/db.py](file://letta/server/db.py)

## Troubleshooting Common Issues

This section addresses common database issues encountered in Letta deployments and provides troubleshooting guidance for resolution. The application's architecture includes several mechanisms to help diagnose and resolve database-related problems.

Connection timeouts are a common issue that can occur due to network problems, database overload, or incorrect configuration. The application's connection pool settings in `settings.py` include a `pg_pool_timeout` of 30 seconds, which should be adjusted based on network conditions and database performance. If connection timeouts persist, administrators should verify network connectivity between the application and database containers, check database resource utilization, and consider increasing the pool size or timeout values. The `pool_pre_ping` setting helps detect dead connections, but if issues continue, disabling connection pooling temporarily by setting `disable_sqlalchemy_pooling` to false may help identify the root cause.

Migration conflicts can occur when multiple instances attempt to apply database migrations simultaneously or when migration scripts contain errors. The Alembic migration system maintains a `alembic_version` table to track the current migration state, which can be checked to diagnose conflicts. If migration conflicts occur, administrators should ensure that only one instance applies migrations at a time, typically during deployment. The `env.py` file includes logic to skip migrations when running with SQLite, which can help isolate PostgreSQL-specific migration issues. For persistent migration problems, manually verifying the database schema against the expected state and consulting the migration script history in the `alembic/versions/` directory can help identify and resolve conflicts.

Other common issues include authentication failures, which can be diagnosed by verifying the `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` environment variables in the compose files match the credentials used in the `LETTA_PG_URI`. Performance issues may indicate the need for query optimization, proper indexing, or adjustment of connection pool settings. The application's logging configuration can be enabled to provide detailed information about database operations, helping to identify slow queries or inefficient access patterns. Regular monitoring of database performance metrics and connection pool statistics can help proactively identify and address potential issues before they impact application functionality.

**Section sources**
- [letta/settings.py](file://letta/settings.py)
- [letta/server/db.py](file://letta/server/db.py)
- [alembic/env.py](file://alembic/env.py)
- [compose.yaml](file://compose.yaml)