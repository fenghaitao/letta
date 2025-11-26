# Agent Model

<cite>
**Referenced Files in This Document**   
- [agent.py](file://letta/orm/agent.py)
- [base.py](file://letta/orm/base.py)
- [sqlalchemy_base.py](file://letta/orm/sqlalchemy_base.py)
- [agent.py](file://letta/schemas/agent.py)
- [tools_agents.py](file://letta/orm/tools_agents.py)
- [sources_agents.py](file://letta/orm/sources_agents.py)
- [blocks_agents.py](file://letta/orm/blocks_agents.py)
- [d05669b60ebe_migrate_agents_to_orm.py](file://alembic/versions/d05669b60ebe_migrate_agents_to_orm.py)
- [348214cbc081_add_org_agent_id_indices.py](file://alembic/versions/348214cbc081_add_org_agent_id_indices.py)
- [74e860718e0d_add_archival_memory_sharing.py](file://alembic/versions/74e860718e0d_add_archival_memory_sharing.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Core Fields](#core-fields)
3. [Relationships](#relationships)
4. [Agent State Serialization](#agent-state-serialization)
5. [Indexes and Query Optimization](#indexes-and-query-optimization)
6. [Data Lifecycle and Retention](#data-lifecycle-and-retention)
7. [Common Queries](#common-queries)
8. [Conclusion](#conclusion)

## Introduction

The Agent entity in Letta's database schema represents an autonomous agent that can interact with users, tools, and data sources. This document provides comprehensive documentation of the Agent model, detailing its fields, relationships, state serialization mechanism, indexing strategy, and data lifecycle management.

The Agent model is implemented as a SQLAlchemy ORM class that inherits from `SqlalchemyBase`, `OrganizationMixin`, `ProjectMixin`, `TemplateEntityMixin`, and `TemplateMixin`. It serves as the central entity in the system, connecting various components such as tools, data sources, memory blocks, and messages.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L37-L44)

## Core Fields

The Agent model contains several core fields that define its identity, configuration, and operational state.

### Basic Identification Fields
- **id**: Primary key field that uniquely identifies each agent. The ID is generated automatically using a UUID with the prefix "agent-". This field is defined as `Mapped[str] = mapped_column(String, primary_key=True, default=lambda: f"agent-{uuid.uuid4()}")`.
- **name**: Optional human-readable identifier for the agent. This field is non-unique and can be used for display purposes. Defined as `Mapped[Optional[str]] = mapped_column(String, nullable=True)`.
- **description**: Optional description field that provides additional information about the agent's purpose or functionality.

### System Configuration Fields
- **agent_type**: Optional field that specifies the type of agent using the `AgentType` enum. This determines the agent's behavior and capabilities.
- **system**: Optional field containing the system prompt used by the agent, which defines its personality and operational guidelines.
- **message_ids**: Optional JSON field containing a list of message IDs that are currently in the agent's in-context memory.
- **response_format**: Optional field that specifies the response format for the agent using the `ResponseFormatUnion` type.

### Metadata and Configuration Objects
- **metadata_**: Optional JSON field for storing arbitrary metadata about the agent.
- **llm_config**: Optional field containing the LLM backend configuration object for this agent, stored using a custom `LLMConfigColumn` type.
- **embedding_config**: Field containing the embedding configuration object for this agent, stored using a custom `EmbeddingConfigColumn` type.
- **tool_rules**: Optional field containing a list of tool rules that govern the agent's tool usage, stored using a custom `ToolRulesColumn` type.

### Operational State Fields
- **message_buffer_autoclear**: Boolean field that determines whether the agent will automatically clear its message buffer between interactions.
- **enable_sleeptime**: Optional boolean field that enables background memory management for the agent.
- **last_run_completion**: Optional datetime field that records when the agent last completed a run.
- **last_run_duration_ms**: Optional integer field that records the duration of the agent's last run in milliseconds.
- **last_stop_reason**: Optional field that records the stop reason from the agent's last run using the `StopReasonType` enum.
- **timezone**: Optional string field that specifies the agent's timezone for context window management.
- **max_files_open**: Optional integer field that limits the maximum number of files that can be open simultaneously for this agent.
- **per_file_view_window_char_limit**: Optional integer field that sets the character limit for viewing individual files.
- **hidden**: Optional boolean field that controls whether the agent is visible in listings.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L46-L114)

## Relationships

The Agent model has several relationships with other entities in the system, enabling rich functionality and data connections.

### Tools Relationship
Agents can be associated with multiple tools through the `tools_agents` mapping table. This relationship is defined as:
```python
tools: Mapped[List["Tool"]] = relationship("Tool", secondary="tools_agents", lazy="selectin", passive_deletes=True)
```

The `tools_agents` table serves as a many-to-many mapping between agents and tools, with the following structure:
- **agent_id**: Foreign key referencing the agents table with CASCADE delete
- **tool_id**: Foreign key referencing the tools table with CASCADE delete
- Composite primary key on (agent_id, tool_id)
- Unique constraint on (agent_id, tool_id) named "unique_agent_tool"

This relationship enables agents to leverage various capabilities provided by different tools.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L124)
- [tools_agents.py](file://letta/orm/tools_agents.py#L7-L19)

### Sources Relationship
Agents can access multiple data sources through the `sources_agents` mapping table. This relationship is defined as:
```python
sources: Mapped[List["Source"]] = relationship("Source", secondary="sources_agents", lazy="selectin")
```

The `sources_agents` table serves as a many-to-many mapping between agents and sources, with the following structure:
- **agent_id**: Foreign key referencing the agents table with CASCADE delete
- **source_id**: Foreign key referencing the sources table with CASCADE delete
- Composite primary key on (agent_id, source_id)

This relationship allows agents to retrieve information from various data sources for knowledge retrieval and processing.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L125)
- [sources_agents.py](file://letta/orm/sources_agents.py#L7-L15)

### Blocks (Core Memory) Relationship
Agents maintain core memory through blocks using the `blocks_agents` mapping table. This relationship is defined as:
```python
core_memory: Mapped[List["Block"]] = relationship(
    "Block",
    secondary="blocks_agents",
    lazy="selectin",
    passive_deletes=True,
    back_populates="agents",
    doc="Blocks forming the core memory of the agent."
)
```

The `blocks_agents` table serves as a many-to-many mapping between agents and blocks, with additional complexity due to the composite primary key on the Block entity. The table structure includes:
- **agent_id**: Foreign key referencing the agents table with CASCADE delete
- **block_id**: Part of the composite primary key, referencing the block's ID
- **block_label**: Part of the composite primary key, referencing the block's label
- Unique constraint on (agent_id, block_id) named "unique_agent_block"
- Unique constraint on (agent_id, block_label) named "unique_label_per_agent"
- Foreign key constraint on (block_id, block_label) referencing (block.id, block.label) with CASCADE updates and deletes

This relationship enables agents to maintain structured memory elements that persist across interactions.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L126-L133)
- [blocks_agents.py](file://letta/orm/blocks_agents.py#L7-L35)

### Messages Relationship
While not explicitly shown as a direct relationship in the Agent model, agents are connected to messages through the agent_id field in the messages table. This foreign key relationship allows for efficient retrieval of all messages associated with a particular agent.

The migration file `348214cbc081_add_org_agent_id_indices.py` confirms this relationship by creating an index on the messages table for both organization_id and agent_id:
```python
op.create_index("ix_messages_org_agent", "messages", ["organization_id", "agent_id"], unique=False)
```

This indexing strategy optimizes queries that filter messages by both organization and agent.

**Section sources**
- [348214cbc081_add_org_agent_id_indices.py](file://alembic/versions/348214cbc081_add_org_agent_id_indices.py#L28)

### Other Relationships
The Agent model also has several other important relationships:
- **organization**: Relationship with the Organization model, with the agent belonging to a specific organization
- **tool_exec_environment_variables**: Relationship with environment variables specific to tool execution for this agent
- **tags**: Relationship with tags associated with the agent
- **runs**: Relationship with execution runs associated with the agent
- **identities**: Relationship with identities associated with the agent
- **groups**: Relationship with groups the agent belongs to
- **archives_agents**: Relationship with archives accessible by this agent

These relationships enable comprehensive management of agent context, permissions, and execution history.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L116-L183)

## Agent State Serialization

The Agent model includes sophisticated serialization mechanisms that convert the SQLAlchemy model into its Pydantic counterpart for API exposure and client consumption.

### Serialization Methods
The model implements two primary serialization methods:
- **to_pydantic**: Synchronous method that converts the SQLAlchemy Agent model into its Pydantic representation
- **to_pydantic_async**: Asynchronous method that performs the same conversion, optimized for async operations

Both methods follow a consistent pattern:
1. Extract base fields that are always included
2. Conditionally include optional fields based on the `include_relationships` parameter
3. Transform certain fields for API compatibility (e.g., mapping `metadata_` to `metadata`)

The base fields always included in serialization are:
- id, agent_type, name, description, system, message_ids, metadata_
- llm_config, embedding_config, project_id, template_id, base_template_id
- tool_rules, message_buffer_autoclear, tags

Optional fields that can be included based on the `include_relationships` parameter include:
- tools, sources, memory, blocks, identity_ids, identities
- multi_agent_group, tool_exec_environment_variables, secrets

### State Management
The serialization process handles several important transformations:
- Mapping the internal `metadata_` field to the external `metadata` field in the Pydantic model
- Converting the `llm_config` object to extract the model handle and model settings
- Converting the `embedding_config` object to extract the embedding handle
- Calculating derived fields like the per-file view window character limit based on context window size

The `to_pydantic_async` method is particularly optimized for performance, using `awaitable_attrs` to efficiently load only the requested relationships and avoiding unnecessary database queries.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L194-L435)

## Indexes and Query Optimization

The Agent model and related tables include several indexes designed to optimize common query patterns and ensure efficient data retrieval.

### Agent Table Indexes
The Agent table has three composite indexes defined in its `__table_args__`:
- **ix_agents_created_at**: Index on (created_at, id) for efficient chronological queries
- **ix_agents_organization_id_deployment_id**: Index on (organization_id, deployment_id) for filtering agents by organization and deployment
- **ix_agents_project_id**: Index on project_id for efficient queries by project

These indexes support common access patterns such as retrieving agents in chronological order, filtering by organization, or querying by project affiliation.

### Mapping Table Indexes
The relationship mapping tables include indexes optimized for join operations:

#### tools_agents Table
- **ix_tools_agents_tool_id**: Index on tool_id for efficient queries of which agents use a specific tool

#### sources_agents Table
- **ix_sources_agents_source_id**: Index on source_id for efficient queries of which agents access a specific source

#### blocks_agents Table
- **ix_blocks_agents_block_label_agent_id**: Index on (block_label, agent_id) for efficient queries by block label and agent
- **ix_blocks_agents_block_id**: Index on block_id for efficient queries of which agents use a specific block

### Cross-Table Indexes
Additional indexes have been added to optimize queries across multiple tables:
- **ix_messages_org_agent**: Index on (organization_id, agent_id) in the messages table for efficient retrieval of messages by organization and agent
- **ix_agent_passages_org_agent**: Index on (organization_id, agent_id) in the agent_passages table (now migrated) for efficient retrieval of agent passages

These indexing strategies ensure that common query patterns involving agents and their related entities perform efficiently, even as data volumes grow.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L40-L44)
- [tools_agents.py](file://letta/orm/tools_agents.py#L13)
- [sources_agents.py](file://letta/orm/sources_agents.py#L11)
- [blocks_agents.py](file://letta/orm/blocks_agents.py#L27-L28)
- [348214cbc081_add_org_agent_id_indices.py](file://alembic/versions/348214cbc081_add_org_agent_id_indices.py#L27-L28)

## Data Lifecycle and Retention

The Agent model and its related entities follow a comprehensive data lifecycle management strategy with well-defined retention policies and cascading operations.

### Cascading Deletions
The model implements cascading deletions to maintain data integrity when agents are removed:
- **tools_agents**: When an agent is deleted, the corresponding entries in the tools_agents mapping table are automatically removed due to the CASCADE delete constraint on the foreign key
- **sources_agents**: Similarly, entries in the sources_agents mapping table are automatically removed when an agent is deleted
- **blocks_agents**: Entries in the blocks_agents mapping table are automatically removed when an agent is deleted
- **tool_exec_environment_variables**: This relationship has a cascade="all, delete-orphan" setting, ensuring that environment variables specific to the agent are deleted when the agent is removed
- **runs**: The runs relationship also has cascade="all, delete-orphan", ensuring that all execution runs associated with the agent are deleted
- **file_agents**: The file_agents relationship has cascade="all, delete-orphan", ensuring that file associations are cleaned up
- **archives_agents**: The archives_agents relationship has cascade="all, delete-orphan", ensuring that archive access permissions are removed

This cascading deletion strategy ensures that when an agent is deleted, all related data is properly cleaned up, preventing orphaned records and maintaining database integrity.

### Soft Delete Mechanism
The Agent model inherits from `CommonSqlalchemyMetaMixins` which includes a soft delete mechanism:
- **is_deleted**: Boolean field with a server default of FALSE that marks records as deleted without removing them from the database
- When an agent is "deleted," the `is_deleted` field is set to TRUE rather than the record being physically removed
- This allows for data recovery and maintains historical records while making the agent effectively invisible to normal queries

The `delete_async` method implements this soft delete behavior:
```python
async def delete_async(self, db_session: "AsyncSession", actor: Optional["User"] = None) -> "SqlalchemyBase":
    self.is_deleted = True
    return await self.update_async(db_session)
```

### Hard Delete Option
For cases where permanent removal is required, the model provides a `hard_delete_async` method that physically removes the record from the database:
```python
async def hard_delete_async(self, db_session: "AsyncSession", actor: Optional["User"] = None) -> None:
    await db_session.delete(self)
    await db_session.commit()
```

This method should be used with caution as it permanently removes the agent and all cascading relationships.

### Data Retention Considerations
The migration file `74e860718e0d_add_archival_memory_sharing.py` demonstrates a significant data retention strategy change:
- Previously, agent passages were stored directly in the agent_passages table linked to agents
- This was migrated to a shared archive system where passages are stored in archives that can be shared among multiple agents
- This change enables better data retention and sharing capabilities, allowing archival memory to persist even if individual agents are deleted

This architectural decision reflects a thoughtful approach to data lifecycle management, balancing the need for agent isolation with the benefits of shared knowledge resources.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L117-L183)
- [d05669b60ebe_migrate_agents_to_orm.py](file://alembic/versions/d05669b60ebe_migrate_agents_to_orm.py#L89-L92)
- [74e860718e0d_add_archival_memory_sharing.py](file://alembic/versions/74e860718e0d_add_archival_memory_sharing.py#L131-L163)

## Common Queries

This section documents common query patterns for retrieving agents and related data.

### Retrieving Agents by Project
To retrieve all agents belonging to a specific project, use the project_id field:
```python
agents = await Agent.list_async(db_session=session, project_id="project-123")
```

This query leverages the `ix_agents_project_id` index for efficient filtering.

### Retrieving Agents by Organization
To retrieve all agents within a specific organization, the system automatically applies organization-based access control:
```python
agents = await Agent.list_async(db_session=session, actor=user)
```

When an actor (user) is provided, the query automatically filters to only include agents from the same organization as the actor, leveraging the `ix_agents_organization_id_deployment_id` index.

### Retrieving Agents with Specific Tools
To find agents that use a specific tool, query through the tools_agents mapping table:
```python
agents = await Agent.list_async(
    db_session=session,
    join_model=Tool,
    join_conditions=[Agent.tools.any(Tool.id == "tool-123")]
)
```

This leverages the `ix_tools_agents_tool_id` index for efficient lookups.

### Retrieving Agents with Specific Sources
To find agents that access a specific data source:
```python
agents = await Agent.list_async(
    db_session=session,
    join_model=Source,
    join_conditions=[Agent.sources.any(Source.id == "source-123")]
)
```

This leverages the `ix_sources_agents_source_id` index for efficient lookups.

### Retrieving Agents with Specific Memory Blocks
To find agents that use a specific memory block:
```python
agents = await Agent.list_async(
    db_session=session,
    join_model=Block,
    join_conditions=[Agent.core_memory.any(Block.id == "block-123")]
)
```

This leverages the `ix_blocks_agents_block_id` index for efficient lookups.

### Chronological Agent Retrieval
To retrieve agents in chronological order:
```python
agents = await Agent.list_async(
    db_session=session,
    ascending=False  # Most recent first
)
```

This leverages the `ix_agents_created_at` index for efficient sorting.

### Text Search on Agent Names
The system supports text search on agent names through the `_list_preprocess` method:
```python
agents = await Agent.list_async(
    db_session=session,
    query_text="sales"
)
```

This will find agents with "sales" in their name by applying a case-insensitive contains filter.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L290-L295)
- [sqlalchemy_base.py](file://letta/orm/sqlalchemy_base.py#L200-L202)

## Conclusion

The Agent model in Letta's database schema is a comprehensive entity that serves as the central component of the system. It represents autonomous agents with rich capabilities for interacting with users, tools, and data sources.

Key aspects of the Agent model include:
- Comprehensive field definitions that capture agent identity, configuration, and operational state
- Sophisticated relationships with tools, data sources, memory blocks, and other entities
- Efficient serialization mechanisms for API exposure
- Optimized indexing strategies for common query patterns
- Robust data lifecycle management with cascading deletions and soft delete capabilities

The model's design reflects careful consideration of performance, data integrity, and usability requirements. The use of mapping tables for many-to-many relationships, cascading deletions for data integrity, and strategic indexing for query optimization demonstrate a mature database design approach.

The Agent model serves as the foundation for Letta's agent-based architecture, enabling the creation of intelligent, autonomous agents that can perform complex tasks while maintaining persistent state and knowledge.

**Section sources**
- [agent.py](file://letta/orm/agent.py#L37-L436)
- [base.py](file://letta/orm/base.py#L1-L86)
- [sqlalchemy_base.py](file://letta/orm/sqlalchemy_base.py#L1-L816)