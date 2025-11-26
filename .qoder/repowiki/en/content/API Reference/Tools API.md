# Tools API

<cite>
**Referenced Files in This Document**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py)
- [tool.py](file://letta/schemas/tool.py)
- [tool_manager.py](file://letta/services/tool_manager.py)
- [enums.py](file://letta/schemas/enums.py)
- [tool.py](file://letta/orm/tool.py)
- [mcp_oauth.py](file://letta/orm/mcp_oauth.py)
- [mcp.py](file://letta/schemas/mcp.py)
- [mcp_server.py](file://letta/schemas/mcp_server.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Authentication](#authentication)
3. [API Endpoints Overview](#api-endpoints-overview)
4. [Tool Management Endpoints](#tool-management-endpoints)
5. [Tool Schema Definition](#tool-schema-definition)
6. [Query Parameters and Filtering](#query-parameters-and-filtering)
7. [Pagination and Sorting](#pagination-and-sorting)
8. [Error Codes](#error-codes)
9. [Relationship Between Tools and Agents](#relationship-between-tools-and-agents)
10. [Examples and Usage](#examples-and-usage)

## Introduction

The Tools API provides comprehensive REST endpoints for managing tools in the Letta platform. Tools are functions that can be called by agents to perform various operations, ranging from built-in core functions to custom user-defined tools and external integrations via MCP (Model Context Protocol).

The API supports CRUD operations for tools, including creation, retrieval, listing, updating, and deletion. It also provides specialized endpoints for MCP server management, tool generation, and execution.

## Authentication

All tool management endpoints require authentication using Bearer tokens. The API expects the Authorization header to be set with a valid Bearer token:

```http
Authorization: Bearer YOUR_API_TOKEN
```

The token must belong to a user who has access to the organization's tools. The system automatically associates tools with the user's organization for access control.

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L50-L62)

## API Endpoints Overview

The Tools API consists of several main categories of endpoints:

### Core Tool Operations
- **GET /** - List and filter tools with pagination
- **GET /{tool_id}** - Retrieve a specific tool
- **POST /** - Create a new tool
- **PUT /** - Create or update a tool (upsert)
- **PATCH /{tool_id}** - Update an existing tool
- **DELETE /{tool_id}** - Delete a tool

### Tool Information
- **GET /count** - Get count of tools matching filters
- **POST /run** - Execute a tool from source code
- **POST /generate-tool** - Generate a tool from a prompt

### MCP Server Management
- **GET /mcp/servers** - List MCP servers
- **GET /mcp/servers/{server_name}/tools** - List tools from MCP server
- **POST /mcp/servers/{server_name}/resync** - Resync MCP server tools
- **POST /mcp/servers/{server_name}/{tool_name}** - Add MCP tool
- **POST /mcp/servers** - Add MCP server
- **PATCH /mcp/servers/{server_name}** - Update MCP server
- **DELETE /mcp/servers/{server_name}** - Delete MCP server

### OAuth and Authentication
- **POST /mcp/servers/connect** - Connect to MCP server with OAuth
- **GET /mcp/oauth/callback/{session_id}** - Handle OAuth callback

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L171-L900)

## Tool Management Endpoints

### List Tools

**Endpoint:** `GET /tools`

Retrieve a list of tools with optional filtering and pagination.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `before` | string | Tool ID cursor for pagination. Returns tools that come before this tool ID in the specified sort order |
| `after` | string | Tool ID cursor for pagination. Returns tools that come after this tool ID in the specified sort order |
| `limit` | integer | Maximum number of tools to return (default: 50) |
| `order` | string | Sort order for tools by creation time ('asc' for oldest first, 'desc' for newest first, default: 'desc') |
| `order_by` | string | Field to sort by (default: 'created_at') |
| `name` | string | Filter by single tool name |
| `names` | array[string] | Filter by specific tool names |
| `tool_ids` | array[string] | Filter by specific tool IDs |
| `search` | string | Search tool names (case-insensitive partial match) |
| `tool_types` | array[string] | Filter by tool type(s) |
| `exclude_tool_types` | array[string] | Tool type(s) to exclude |
| `return_only_letta_tools` | boolean | Return only tools with tool_type starting with 'letta_' |

**Example Request:**
```bash
curl -X GET "https://api.letta.com/tools?limit=10&order=desc&tool_types=custom,memory" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Example Response:**
```json
[
  {
    "id": "tool_123",
    "name": "send_email",
    "description": "Send an email to a recipient",
    "tool_type": "custom",
    "source_type": "python",
    "tags": ["communication", "email"],
    "created_by_id": "user_456",
    "created_at": "2024-01-15T10:30:00Z",
    "metadata_": {}
  }
]
```

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L171-L271)

### Retrieve Tool

**Endpoint:** `GET /tools/{tool_id}`

Get a specific tool by its ID.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tool_id` | string | The unique identifier of the tool |

**Example Request:**
```bash
curl -X GET "https://api.letta.com/tools/tool_123" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Example Response:**
```json
{
  "id": "tool_123",
  "name": "calculate_discount",
  "description": "Calculate discount amount based on percentage and original price",
  "tool_type": "custom",
  "source_type": "python",
  "source_code": "def calculate_discount(price: float, discount_percent: float) -> float:\n    return price * (discount_percent / 100)",
  "json_schema": {
    "name": "calculate_discount",
    "description": "Calculate discount amount based on percentage and original price",
    "parameters": {
      "type": "object",
      "properties": {
        "price": {"type": "number", "description": "Original price"},
        "discount_percent": {"type": "number", "description": "Discount percentage"}
      },
      "required": ["price", "discount_percent"]
    }
  },
  "tags": ["math", "finance"],
  "created_by_id": "user_456",
  "created_at": "2024-01-15T10:30:00Z",
  "metadata_": {}
}
```

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L154-L168)

### Create Tool

**Endpoint:** `POST /tools`

Create a new tool with the provided configuration.

**Request Body:**
```json
{
  "name": "new_tool",
  "description": "Tool description",
  "source_code": "def new_tool():\n    return 'Hello World'",
  "source_type": "python",
  "json_schema": {
    "name": "new_tool",
    "description": "Tool description",
    "parameters": {
      "type": "object",
      "properties": {},
      "required": []
    }
  },
  "tags": ["utility"],
  "return_char_limit": 1000,
  "pip_requirements": [],
  "npm_requirements": [],
  "default_requires_approval": false,
  "enable_parallel_execution": false
}
```

**Response:**
Returns the created tool object with all fields populated.

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L274-L288)

### Update Tool

**Endpoint:** `PATCH /tools/{tool_id}`

Update an existing tool with the provided fields.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tool_id` | string | The unique identifier of the tool |

**Request Body:**
```json
{
  "description": "Updated description",
  "source_code": "def new_tool():\n    return 'Updated Hello World'",
  "tags": ["utility", "updated"],
  "default_requires_approval": true
}
```

**Response:**
Returns the updated tool object.

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L308-L323)

### Delete Tool

**Endpoint:** `DELETE /tools/{tool_id}`

Delete a tool by its ID.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tool_id` | string | The unique identifier of the tool |

**Response:**
Returns a success response with no body.

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L51-L62)

### Count Tools

**Endpoint:** `GET /tools/count`

Get the count of tools matching the specified filters.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | string | Filter by single tool name |
| `names` | array[string] | Filter by specific tool names |
| `tool_ids` | array[string] | Filter by specific tool IDs |
| `search` | string | Search tool names (case-insensitive partial match) |
| `tool_types` | array[string] | Filter by tool type(s) |
| `exclude_tool_types` | array[string] | Tool type(s) to exclude |
| `return_only_letta_tools` | boolean | Count only tools with tool_type starting with 'letta_' |
| `exclude_letta_tools` | boolean | Exclude built-in Letta tools from the count |

**Example Request:**
```bash
curl -X GET "https://api.letta.com/tools/count?tool_types=custom&search=email" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Example Response:**
```json
15
```

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L64-L151)

## Tool Schema Definition

### Core Tool Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier for the tool |
| `name` | string | Display name of the tool |
| `description` | string | Description of what the tool does |
| `tool_type` | string | Type of tool (see ToolType enumeration) |
| `source_type` | string | Type of source code (python, typescript, json) |
| `source_code` | string | The actual source code of the function |
| `json_schema` | object | OpenAI-compatible JSON schema of the function |
| `args_json_schema` | object | JSON schema of the function arguments |
| `tags` | array[string] | Metadata tags for filtering tools |
| `return_char_limit` | integer | Maximum number of characters the tool can return |
| `pip_requirements` | array | Optional list of pip packages required by this tool |
| `npm_requirements` | array | Optional list of npm packages required by this tool |
| `default_requires_approval` | boolean | Whether tool execution requires approval |
| `enable_parallel_execution` | boolean | Whether tool can be executed concurrently |
| `created_by_id` | string | ID of the user who created the tool |
| `last_updated_by_id` | string | ID of the user who last updated the tool |
| `metadata_` | object | Dictionary of additional metadata for the tool |

### Tool Types

The `tool_type` field determines how the tool behaves and where it's sourced from:

| Type | Description |
|------|-------------|
| `custom` | User-created tools with custom source code |
| `letta_core` | Built-in Letta core tools |
| `letta_memory_core` | Memory-related core tools |
| `letta_multi_agent_core` | Multi-agent coordination tools |
| `letta_sleeptime_core` | Sleep-time related tools |
| `letta_voice_sleeptime_core` | Voice and sleep-time tools |
| `letta_builtin` | Legacy built-in tools |
| `letta_files_core` | File management tools |
| `external_mcp` | Tools from Model Context Protocol servers |

**Section sources**
- [tool.py](file://letta/schemas/tool.py#L35-L70)
- [enums.py](file://letta/schemas/enums.py#L194-L207)

## Query Parameters and Filtering

### Basic Filtering

**Single Name Filter:**
```bash
curl "https://api.letta.com/tools?name=send_email"
```

**Multiple Names Filter:**
```bash
curl "https://api.letta.com/tools?names=send_email,get_weather&names=calculate_discount"
```

**Tool IDs Filter:**
```bash
curl "https://api.letta.com/tools?tool_ids=tool_123,tool_456,tool_789"
```

### Advanced Filtering

**Tool Type Filtering:**
```bash
curl "https://api.letta.com/tools?tool_types=custom,memory"
```

**Exclusion Filtering:**
```bash
curl "https://api.letta.com/tools?exclude_tool_types=builtin,composio"
```

**Search Filtering:**
```bash
curl "https://api.letta.com/tools?search=email&tool_types=custom"
```

### Combined Filters

```bash
curl "https://api.letta.com/tools?tool_types=custom&search=weather&limit=20"
```

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L184-L193)

## Pagination and Sorting

### Cursor-Based Pagination

The API uses cursor-based pagination for efficient large dataset navigation:

**Basic Pagination:**
```bash
# Get first page
curl "https://api.letta.com/tools?limit=50"

# Get next page using after cursor
curl "https://api.letta.com/tools?after=tool_100&limit=50"

# Get previous page using before cursor
curl "https://api.letta.com/tools?before=tool_200&limit=50"
```

**Window-Based Pagination:**
```bash
# Get tools between two cursors
curl "https://api.letta.com/tools?after=tool_100&before=tool_300&limit=100"
```

### Sorting Options

**Sort Order:**
```bash
# Newest first (default)
curl "https://api.letta.com/tools?order=desc"

# Oldest first
curl "https://api.letta.com/tools?order=asc"
```

**Sort Field:**
```bash
# Sort by creation time (default)
curl "https://api.letta.com/tools?order_by=created_at"

# Future: Sort by other fields (if supported)
```

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L173-L182)
- [tool_manager.py](file://letta/services/tool_manager.py#L459-L615)

## Error Codes

### HTTP Status Codes

| Status Code | Description | Common Causes |
|-------------|-------------|---------------|
| 200 | OK | Successful request |
| 400 | Bad Request | Invalid parameters, malformed JSON |
| 401 | Unauthorized | Invalid or missing Bearer token |
| 404 | Not Found | Tool ID not found |
| 409 | Conflict | Tool name conflict |
| 422 | Unprocessable Entity | Validation errors |
| 500 | Internal Server Error | Server-side error |

### Tool-Specific Errors

**Tool Creation Errors:**
- **Duplicate Tool Name**: Tool with the same name already exists in the organization
- **Schema Mismatch**: Tool name doesn't match JSON schema name
- **Invalid Source Code**: Syntax errors in the provided source code
- **Invalid JSON Schema**: Malformed or invalid JSON schema

**Tool Execution Errors:**
- **Tool Not Found**: Specified tool ID doesn't exist
- **Permission Denied**: Insufficient permissions to access the tool
- **Execution Timeout**: Tool execution exceeded timeout limits
- **Resource Limits**: Tool exceeded character or memory limits

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L280-L288)
- [tool_manager.py](file://letta/services/tool_manager.py#L200-L255)

## Relationship Between Tools and Agents

Tools are fundamental components that enable agents to perform actions. The relationship between tools and agents is managed through several mechanisms:

### Tool Assignment

Agents can be configured with specific tools from the available tool library. The assignment process involves:

1. **Tool Discovery**: Agents discover available tools through the tool management API
2. **Tool Selection**: Agents select tools based on their capabilities and current needs
3. **Tool Invocation**: Agents call tools during their reasoning loops

### Tool Categories for Agents

Different tool types serve different agent capabilities:

**Core Tools:**
- Memory management tools for agent memory operations
- Communication tools for external messaging
- File management tools for document handling

**Custom Tools:**
- User-defined tools tailored to specific use cases
- Domain-specific functionality
- Integration with external systems

**External Tools (MCP):**
- Third-party service integrations
- Specialized tool sets from MCP servers
- Cloud service APIs

### Tool Lifecycle Management

The tool lifecycle directly impacts agent functionality:

**Creation Impact:**
- New tools become immediately available to agents
- Agents can be configured to use new tools automatically

**Update Impact:**
- Updated tools affect all agents using that tool
- Changes propagate to agent configurations

**Deletion Impact:**
- Removed tools are no longer available to agents
- Agents using deleted tools may fail until reconfigured

**Section sources**
- [tool_manager.py](file://letta/services/tool_manager.py#L459-L516)

## Examples and Usage

### Creating a Custom Python Tool

**Example: Email Sending Tool**

```bash
curl -X POST "https://api.letta.com/tools" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "send_email",
    "description": "Send an email to a recipient",
    "source_code": "def send_email(recipient: str, subject: str, body: str):\n    """Send an email to the specified recipient."""\n    print(f\"Sending email to {recipient}\\nSubject: {subject}\\nBody: {body}\")\n    return \"Email sent successfully\"",
    "source_type": "python",
    "tags": ["communication", "email"],
    "return_char_limit": 1000,
    "default_requires_approval": false
  }'
```

### Listing Tools with Type Filtering

**Example: Get All Custom Tools**

```bash
curl -X GET "https://api.letta.com/tools?tool_types=custom&limit=20" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Example: Search Tools by Name**

```bash
curl -X GET "https://api.letta.com/tools?search=weather&tool_types=custom" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Managing MCP Tools

**Example: Add MCP Server**

```bash
curl -X POST "https://api.letta.com/tools/mcp/servers" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "server_name": "weather-service",
    "server_url": "https://weather-api.example.com",
    "auth_header": "Authorization",
    "auth_token": "Bearer YOUR_API_KEY"
  }'
```

**Example: Add MCP Tool**

```bash
curl -X POST "https://api.letta.com/tools/mcp/servers/weather-service/get_weather" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Executing Tools from Source

**Example: Run Tool Without Creating It**

```bash
curl -X POST "https://api.letta.com/tools/run" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "source_code": "def greet(name: str) -> str:\n    return f\"Hello, {name}!\"",
    "args": {"name": "World"},
    "name": "greet"
  }'
```

### Tool Generation from Prompt

**Example: Generate Tool from Description**

```bash
curl -X POST "https://api.letta.com/tools/generate-tool" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tool_name": "format_currency",
    "prompt": "Create a tool that formats currency amounts with proper decimal places and currency symbol",
    "handle": "gpt-4"
  }'
```

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L274-L288)
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L338-L359)
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L820-L900)