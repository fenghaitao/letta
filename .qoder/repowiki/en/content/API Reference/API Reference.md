# API Reference

<cite>
**Referenced Files in This Document**   
- [app.py](file://letta/server/rest_api/app.py)
- [agents.py](file://letta/server/rest_api/routers/v1/agents.py)
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py)
- [sources.py](file://letta/server/rest_api/routers/v1/sources.py)
- [chat_completions.py](file://letta/server/rest_api/routers/v1/chat_completions.py)
- [jobs.py](file://letta/server/rest_api/routers/v1/jobs.py)
- [agent.py](file://letta/schemas/agent.py)
- [tool.py](file://letta/schemas/tool.py)
- [source.py](file://letta/schemas/source.py)
- [job.py](file://letta/schemas/job.py)
- [message.py](file://letta/schemas/message.py)
- [interface.py](file://letta/server/ws_api/interface.py)
- [protocol.py](file://letta/server/ws_api/protocol.py)
- [agent.py](file://letta/server/rest_api/routers/v1/agents.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Authentication](#authentication)
3. [REST API](#rest-api)
   - [Agents](#agents)
   - [Tools](#tools)
   - [Sources](#sources)
   - [Chat Completions](#chat-completions)
   - [Jobs](#jobs)
4. [WebSocket API](#websocket-api)
5. [Schema Definitions](#schema-definitions)
6. [Client Integration](#client-integration)
7. [Error Handling](#error-handling)
8. [Rate Limiting and Pagination](#rate-limiting-and-pagination)

## Introduction
The Letta platform provides a comprehensive API for creating and managing AI agents with long-term memory and custom tools. This documentation covers both REST and WebSocket APIs for interacting with the platform. The API enables developers to create agents, manage tools and data sources, send messages, and receive real-time responses through streaming endpoints.

The platform supports two main interaction patterns: RESTful operations for CRUD operations and configuration, and WebSocket connections for real-time, bidirectional communication with agents. All endpoints require Bearer token authentication and follow standard HTTP conventions for status codes and error handling.

## Authentication
All API endpoints require authentication using Bearer tokens. Clients must include an Authorization header with each request:

```http
Authorization: Bearer <your-api-key>
```

The API key should be generated through the platform's authentication system and passed as a Bearer token in the Authorization header. Without valid authentication, all requests will receive a 401 Unauthorized response.

**Section sources**
- [app.py](file://letta/server/rest_api/app.py#L70-L78)

## REST API

### Agents
The agents endpoint provides CRUD operations for managing AI agents. Agents are the core entities in the Letta platform, representing persistent AI assistants with memory and tool access.

#### List Agents
Retrieve a list of all agents with optional filtering.

**Endpoint**
```
GET /api/agents
```

**Query Parameters**
| Parameter | Type | Required | Description |
|---------|------|---------|-------------|
| name | string | No | Filter by agent name |
| tags | array[string] | No | Filter by tags |
| match_all_tags | boolean | No | If true, agents must match all tags |
| project_id | string | No | Filter by project ID |
| limit | integer | No | Maximum number of results (default: 50) |
| order | string | No | Sort order: "asc" or "desc" (default: "desc") |
| order_by | string | No | Field to sort by: "created_at" or "last_run_completion" |

**Response**
```json
[
  {
    "id": "agent-123",
    "name": "Research Assistant",
    "agent_type": "memgpt_agent",
    "model": "gpt-4-turbo",
    "tags": ["research", "assistant"],
    "created_at": "2024-01-01T00:00:00Z"
  }
]
```

#### Create Agent
Create a new agent with specified configuration.

**Endpoint**
```
POST /api/agents
```

**Request Body**
```json
{
  "name": "Research Assistant",
  "agent_type": "memgpt_agent",
  "model": "gpt-4-turbo",
  "tools": ["tool-123", "tool-456"],
  "tags": ["research", "assistant"]
}
```

**Response**
Returns the created AgentState object with assigned ID.

#### Get Agent
Retrieve a specific agent by ID.

**Endpoint**
```
GET /api/agents/{agent_id}
```

**Response**
Returns the AgentState object for the specified agent.

#### Update Agent
Modify an existing agent's configuration.

**Endpoint**
```
PATCH /api/agents/{agent_id}
```

**Request Body**
```json
{
  "name": "Updated Research Assistant",
  "tags": ["research", "updated"]
}
```

**Response**
Returns the updated AgentState object.

#### Delete Agent
Remove an agent from the system.

**Endpoint**
```
DELETE /api/agents/{agent_id}
```

**Response**
204 No Content on success.

**Section sources**
- [agents.py](file://letta/server/rest_api/routers/v1/agents.py#L77-L166)
- [agent.py](file://letta/schemas/agent.py#L60-L200)

### Tools
The tools endpoint manages custom functions that agents can execute. Tools extend agent capabilities by providing access to external systems and specialized functionality.

#### List Tools
Retrieve available tools with filtering options.

**Endpoint**
```
GET /api/tools
```

**Query Parameters**
| Parameter | Type | Required | Description |
|---------|------|---------|-------------|
| name | string | No | Filter by tool name |
| tool_types | array[string] | No | Filter by tool type |
| search | string | No | Search tool names |
| limit | integer | No | Maximum number of results (default: 50) |

**Response**
```json
[
  {
    "id": "tool-123",
    "name": "search_web",
    "description": "Search the web for information",
    "tool_type": "custom",
    "json_schema": {
      "name": "search_web",
      "description": "Search the web for information",
      "parameters": {
        "type": "object",
        "properties": {
          "query": {
            "type": "string",
            "description": "Search query"
          }
        },
        "required": ["query"]
      }
    }
  }
]
```

#### Create Tool
Register a new tool that agents can use.

**Endpoint**
```
POST /api/tools
```

**Request Body**
```json
{
  "source_code": "def search_web(query: str) -> str:\n    \"\"\"Search the web for information\"\"\"\n    # Implementation here\n    return results",
  "json_schema": {
    "name": "search_web",
    "description": "Search the web for information",
    "parameters": {
      "type": "object",
      "properties": {
        "query": {
          "type": "string",
          "description": "Search query"
        }
      },
      "required": ["query"]
    }
  },
  "description": "Search the web for information",
  "tags": ["web", "search"]
}
```

**Response**
Returns the created Tool object with assigned ID.

#### Get Tool
Retrieve a specific tool by ID.

**Endpoint**
```
GET /api/tools/{tool_id}
```

**Response**
Returns the Tool object for the specified tool.

#### Delete Tool
Remove a tool from the system.

**Endpoint**
```
DELETE /api/tools/{tool_id}
```

**Response**
204 No Content on success.

**Section sources**
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L51-L200)
- [tool.py](file://letta/schemas/tool.py#L35-L200)

### Sources
The sources endpoint manages data sources that provide context and information to agents. Sources contain files and passages that can be retrieved and used during agent operation.

#### List Sources
Retrieve all data sources.

**Endpoint**
```
GET /api/sources
```

**Response**
```json
[
  {
    "id": "source-123",
    "name": "Research Papers",
    "description": "Collection of academic papers",
    "created_at": "2024-01-01T00:00:00Z"
  }
]
```

#### Create Source
Create a new data source.

**Endpoint**
```
POST /api/sources
```

**Request Body**
```json
{
  "name": "Research Papers",
  "description": "Collection of academic papers"
}
```

**Response**
Returns the created Source object with assigned ID.

#### Get Source
Retrieve a specific source by ID.

**Endpoint**
```
GET /api/sources/{source_id}
```

**Response**
Returns the Source object for the specified source.

#### Delete Source
Remove a source and its associated data.

**Endpoint**
```
DELETE /api/sources/{source_id}
```

**Response**
204 No Content on success.

**Section sources**
- [sources.py](file://letta/server/rest_api/routers/v1/sources.py#L50-L180)
- [source.py](file://letta/schemas/source.py#L26-L71)

### Chat Completions
The chat completions endpoint provides OpenAI-compatible interface for interacting with agents. This endpoint supports streaming responses for real-time interaction.

#### Create Chat Completion
Send a message to an agent and receive a response.

**Endpoint**
```
POST /api/chat/completions
```

**Request Body**
```json
{
  "model": "agent-123",
  "messages": [
    {
      "role": "user",
      "content": "What can you help me with?"
    }
  ],
  "stream": true
}
```

**Query Parameters**
| Parameter | Type | Required | Description |
|---------|------|---------|-------------|
| stream | boolean | No | If true, stream responses (default: false) |

**Response**
When streaming is disabled, returns a standard ChatCompletion object. When streaming is enabled, returns a Server-Sent Events stream with ChatCompletionChunk objects.

**Curl Example**
```bash
curl -X POST https://api.letta.com/api/chat/completions \
  -H "Authorization: Bearer <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agent-123",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": true
  }'
```

**Section sources**
- [chat_completions.py](file://letta/server/rest_api/routers/v1/chat_completions.py#L118-L147)

### Jobs
The jobs endpoint manages asynchronous operations such as data processing and batch operations.

#### List Jobs
Retrieve all jobs with optional filtering.

**Endpoint**
```
GET /api/jobs
```

**Query Parameters**
| Parameter | Type | Required | Description |
|---------|------|---------|-------------|
| active | boolean | No | If true, only return active jobs |
| limit | integer | No | Maximum number of results (default: 100) |
| order | string | No | Sort order: "asc" or "desc" (default: "desc") |

**Response**
```json
[
  {
    "id": "job-123",
    "status": "completed",
    "created_at": "2024-01-01T00:00:00Z",
    "completed_at": "2024-01-01T00:01:00Z"
  }
]
```

#### Get Job
Retrieve a specific job by ID.

**Endpoint**
```
GET /api/jobs/{job_id}
```

**Response**
Returns the Job object for the specified job.

#### Cancel Job
Cancel a running job.

**Endpoint**
```
PATCH /api/jobs/{job_id}/cancel
```

**Response**
Returns the updated Job object with cancelled status.

#### Delete Job
Remove a job from the system.

**Endpoint**
```
DELETE /api/jobs/{job_id}
```

**Response**
204 No Content on success.

**Section sources**
- [jobs.py](file://letta/server/rest_api/routers/v1/jobs.py#L16-L143)
- [job.py](file://letta/schemas/job.py#L46-L120)

## WebSocket API
The WebSocket API enables real-time, bidirectional communication with agents. This is ideal for applications requiring low-latency interaction and streaming responses.

### Connection
Establish a WebSocket connection to the server.

**Endpoint**
```
ws://localhost:8283/ws
```

Clients should maintain a persistent connection to receive real-time updates from agents.

### Message Format
All messages are JSON objects with a "type" field indicating the message type.

#### Client to Server
```json
{
  "type": "user_message",
  "message": "Hello agent",
  "agent_id": "agent-123"
}
```

#### Server to Client
The server sends events with different message types:

```json
{
  "type": "agent_response",
  "message_type": "internal_monologue",
  "message": "Thinking about how to respond..."
}
```

```json
{
  "type": "agent_response",
  "message_type": "assistant_message",
  "message": "Hello! How can I help you?"
}
```

```json
{
  "type": "agent_response",
  "message_type": "function_message",
  "message": {
    "function": "search_web",
    "args": {"query": "current weather"},
    "result": "The current weather is sunny..."
  }
}
```

### Event Types
| Event Type | Direction | Description |
|-----------|----------|-------------|
| user_message | Client → Server | Send a message to an agent |
| agent_response | Server → Client | Generic response container |
| internal_monologue | Server → Client | Agent's internal thinking process |
| assistant_message | Server → Client | Final response to user |
| function_message | Server → Client | Tool execution details |

### Real-time Interaction
The WebSocket API enables real-time interaction patterns:

1. Client sends user_message to agent
2. Server streams internal_monologue events as agent thinks
3. Server streams function_message events as tools are executed
4. Server sends assistant_message with final response
5. Connection remains open for subsequent interactions

**Curl Example**
```bash
# Note: curl doesn't support WebSockets natively
# Use a WebSocket client library instead
wscat -c ws://localhost:8283/ws
> {"type": "user_message", "message": "Hello", "agent_id": "agent-123"}
```

**Section sources**
- [interface.py](file://letta/server/ws_api/interface.py#L8-L108)
- [protocol.py](file://letta/server/ws_api/protocol.py#L6-L101)

## Schema Definitions
This section details the core Pydantic models used throughout the API.

### AgentState
The AgentState model represents the complete state of an agent.

```python
class AgentState(OrmMetadataBase):
    id: str
    name: str
    agent_type: AgentType
    model: Optional[str]
    embedding: Optional[str]
    memory: Memory
    tools: List[Tool]
    sources: List[Source]
    tags: List[str]
    created_at: datetime
    message_buffer_autoclear: bool = False
```

**Section sources**
- [agent.py](file://letta/schemas/agent.py#L60-L200)

### Tool
The Tool model represents a function that an agent can execute.

```python
class Tool(BaseTool):
    id: str
    name: Optional[str]
    description: Optional[str]
    source_code: Optional[str]
    json_schema: Optional[Dict]
    tool_type: ToolType = ToolType.CUSTOM
    tags: List[str] = []
    return_char_limit: int = 512
```

**Section sources**
- [tool.py](file://letta/schemas/tool.py#L35-L200)

### Source
The Source model represents a collection of files and passages.

```python
class Source(BaseSource):
    id: str
    name: str
    description: Optional[str]
    embedding_config: EmbeddingConfig
    created_at: Optional[datetime]
    updated_at: Optional[datetime]
```

**Section sources**
- [source.py](file://letta/schemas/source.py#L26-L71)

### Job
The Job model represents an asynchronous operation.

```python
class Job(JobBase):
    id: str
    status: JobStatus
    created_at: datetime
    completed_at: Optional[datetime]
    stop_reason: Optional[StopReasonType]
    metadata: Optional[dict]
    job_type: JobType
```

**Section sources**
- [job.py](file://letta/schemas/job.py#L46-L120)

### Message
The Message model represents a single message in the conversation history.

```python
class Message(BaseMessage):
    id: str
    role: MessageRole
    content: Union[str, List[LettaMessageContentUnion]]
    created_at: datetime
    agent_id: Optional[str]
    user_id: Optional[str]
```

**Section sources**
- [message.py](file://letta/schemas/message.py#L188-L200)

## Client Integration
This section provides guidelines for implementing clients in Python and TypeScript.

### Python SDK
The Python SDK provides a high-level interface to the Letta API.

```python
from letta import Client

# Initialize client
client = Client(api_key="your-api-key")

# Create an agent
agent = client.agents.create(
    name="Research Assistant",
    model="gpt-4-turbo"
)

# Send a message
response = client.agents.send_message(
    agent_id=agent.id,
    message="Hello"
)

# Stream responses
for chunk in client.agents.send_message_stream(
    agent_id=agent.id,
    message="Tell me about AI"
):
    print(chunk)
```

### TypeScript SDK
The TypeScript SDK provides type-safe access to the API.

```typescript
import { LettaClient } from 'letta-sdk';

// Initialize client
const client = new LettaClient({
  apiKey: 'your-api-key',
  baseURL: 'https://api.letta.com'
});

// Create an agent
const agent = await client.agents.create({
  name: 'Research Assistant',
  model: 'gpt-4-turbo'
});

// Send a message
const response = await client.agents.sendMessage({
  agentId: agent.id,
  message: 'Hello'
});

// Stream responses
const stream = client.agents.sendMessageStream({
  agentId: agent.id,
  message: 'Tell me about AI'
});

for await (const chunk of stream) {
  console.log(chunk);
}
```

### Integration Patterns
Common integration patterns include:

1. **Chat Interface**: Use the chat completions endpoint with streaming for real-time chat applications.
2. **Batch Processing**: Use the jobs endpoint to manage long-running data processing tasks.
3. **Real-time Monitoring**: Use WebSocket connections to monitor agent activity and receive immediate updates.
4. **Tool Integration**: Create custom tools to extend agent capabilities with domain-specific functions.

## Error Handling
The API uses standard HTTP status codes and structured error responses.

### Common Status Codes
| Code | Meaning | Description |
|------|--------|-------------|
| 200 | OK | Successful response |
| 201 | Created | Resource created successfully |
| 204 | No Content | Successful operation with no content |
| 400 | Bad Request | Invalid request parameters |
| 401 | Unauthorized | Authentication required |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Resource conflict |
| 422 | Unprocessable Entity | Validation error |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |

### Error Response Format
```json
{
  "detail": "Error description",
  "trace_id": "unique-identifier-for-debugging"
}
```

For validation errors, additional context may be provided:

```json
{
  "detail": "[{'type': 'string_pattern_mismatch', 'loc': ['path', 'agent_id'], 'msg': 'String should match pattern 'agent-{uuid4}'}]",
  "examples": ["agent-123e4567-e89b-42d3-8456-426614174000"],
  "trace_id": "abc123"
}
```

**Section sources**
- [app.py](file://letta/server/rest_api/app.py#L291-L559)

## Rate Limiting and Pagination
The API implements rate limiting and pagination for efficient resource usage.

### Rate Limiting
The API enforces rate limits to ensure fair usage:

- **REST API**: 100 requests per minute per API key
- **WebSocket**: 10 concurrent connections per API key

Exceeding rate limits returns a 429 Too Many Requests response with retry-after header.

### Pagination
List endpoints support cursor-based pagination for efficient data retrieval.

**Query Parameters**
| Parameter | Type | Description |
|---------|------|-------------|
| before | string | Return items before this cursor |
| after | string | Return items after this cursor |
| limit | integer | Maximum number of items (default: 50) |

**Response Headers**
- `Next-Cursor`: Cursor for next page
- `Prev-Cursor`: Cursor for previous page

This approach ensures consistent pagination even with concurrent modifications.

**Section sources**
- [agents.py](file://letta/server/rest_api/routers/v1/agents.py#L78-L166)
- [tools.py](file://letta/server/rest_api/routers/v1/tools.py#L173-L200)
- [jobs.py](file://letta/server/rest_api/routers/v1/jobs.py#L17-L65)