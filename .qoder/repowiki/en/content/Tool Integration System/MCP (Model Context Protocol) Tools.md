# MCP (Model Context Protocol) Tools

<cite>
**Referenced Files in This Document**
- [mcp.py](file://letta/schemas/mcp.py)
- [mcp_tool_executor.py](file://letta/services/tool_executor/mcp_tool_executor.py)
- [base_client.py](file://letta/services/mcp/base_client.py)
- [sse_client.py](file://letta/services/mcp/sse_client.py)
- [oauth_utils.py](file://letta/services/mcp/oauth_utils.py)
- [schema_validator.py](file://letta/functions/schema_validator.py)
- [mcp_manager.py](file://letta/services/mcp_manager.py)
- [mcp_server_manager.py](file://letta/services/mcp_server_manager.py)
- [mcp_servers.py](file://letta/server/rest_api/routers/v1/mcp_servers.py)
- [tool_schema_generator.py](file://letta/services/tool_schema_generator.py)
- [exceptions.py](file://letta/functions/mcp_client/exceptions.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [MCP Server Registration](#mcp-server-registration)
4. [Tool Discovery and Integration](#tool-discovery-and-integration)
5. [Tool Execution Flow](#tool-execution-flow)
6. [Schema Generation and Validation](#schema-generation-and-validation)
7. [Authentication Mechanisms](#authentication-mechanisms)
8. [Performance Considerations](#performance-considerations)
9. [Error Handling and Troubleshooting](#error-handling-and-troubleshooting)
10. [Best Practices](#best-practices)

## Introduction

The MCP (Model Context Protocol) Tools sub-feature enables Letta to integrate with external tools and services through standardized JSON-RPC over Server-Sent Events (SSE) endpoints. This system allows agents to discover, register, and execute tools from MCP-compliant servers, providing a flexible and extensible way to enhance agent capabilities.

MCP servers expose tools through standardized protocols, enabling seamless integration with various external systems. The implementation supports multiple transport types (SSE, STDIO, Streamable HTTP) and provides robust authentication mechanisms including OAuth and API tokens.

## Architecture Overview

The MCP Tools system follows a layered architecture that separates concerns between server management, tool discovery, execution, and validation:

```mermaid
graph TB
subgraph "Client Layer"
Agent[Agent]
REST[REST API]
end
subgraph "Management Layer"
MCPManager[MCP Manager]
ServerManager[Server Manager]
ToolManager[Tool Manager]
end
subgraph "Transport Layer"
SSEClient[SSE Client]
StdioClient[STDIO Client]
HTTPClient[HTTP Client]
end
subgraph "Validation Layer"
SchemaValidator[Schema Validator]
HealthChecker[Health Checker]
end
subgraph "External Systems"
MCPServer[MCP Server]
OAuthProvider[OAuth Provider]
end
Agent --> REST
REST --> MCPManager
MCPManager --> ServerManager
MCPManager --> ToolManager
ServerManager --> SSEClient
ServerManager --> StdioClient
ServerManager --> HTTPClient
SSEClient --> MCPServer
HTTPClient --> MCPServer
SchemaValidator --> ToolManager
HealthChecker --> ToolManager
OAuthProvider --> SSEClient
```

**Diagram sources**
- [mcp_manager.py](file://letta/services/mcp_manager.py#L1-L50)
- [mcp_server_manager.py](file://letta/services/mcp_server_manager.py#L1-L50)
- [base_client.py](file://letta/services/mcp/base_client.py#L1-L50)

**Section sources**
- [mcp_manager.py](file://letta/services/mcp_manager.py#L1-L100)
- [mcp_server_manager.py](file://letta/services/mcp_server_manager.py#L1-L100)

## MCP Server Registration

### REST API Endpoints

MCP servers are registered and managed through a comprehensive REST API that supports CRUD operations:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/mcp-servers/` | POST | Create a new MCP server |
| `/mcp-servers/` | GET | List all MCP servers |
| `/mcp-servers/{id}` | GET | Retrieve specific MCP server |
| `/mcp-servers/{id}` | PATCH | Update MCP server configuration |
| `/mcp-servers/{id}` | DELETE | Delete MCP server |

### Server Configuration Types

The system supports three transport types for MCP server communication:

#### SSE (Server-Sent Events) Configuration
SSE servers communicate over HTTP connections using Server-Sent Events protocol:

```mermaid
sequenceDiagram
participant Client as Letta Client
participant Server as MCP Server
participant Auth as OAuth Provider
Client->>Server : Establish SSE connection
Client->>Auth : Authenticate (if required)
Auth-->>Client : Access token
Client->>Server : Send authenticated request
Server-->>Client : Tool discovery response
Client->>Server : Execute tool request
Server-->>Client : Tool execution result
```

**Diagram sources**
- [sse_client.py](file://letta/services/mcp/sse_client.py#L24-L47)
- [oauth_utils.py](file://letta/services/mcp/oauth_utils.py#L192-L253)

#### STDIO (Standard Input/Output) Configuration
STDIO servers run as local processes with stdin/stdout communication:

```mermaid
flowchart TD
Start([Start STDIO Client]) --> SpawnProcess[Spawn MCP Process]
SpawnProcess --> SetupPipes[Setup stdin/stdout pipes]
SetupPipes --> InitSession[Initialize JSON-RPC session]
InitSession --> Connected{Connected?}
Connected --> |Yes| ToolDiscovery[Discover Tools]
Connected --> |No| Error[Connection Error]
ToolDiscovery --> ExecuteTool[Execute Tool Requests]
ExecuteTool --> ProcessResult[Process Results]
ProcessResult --> ExecuteTool
Error --> Cleanup[Cleanup Resources]
```

**Diagram sources**
- [base_client.py](file://letta/services/mcp/base_client.py#L29-L53)

#### Streamable HTTP Configuration
HTTP servers provide JSON-RPC endpoints over HTTP with streaming capabilities.

**Section sources**
- [mcp_servers.py](file://letta/server/rest_api/routers/v1/mcp_servers.py#L37-L106)
- [mcp.py](file://letta/schemas/mcp.py#L26-L147)

## Tool Discovery and Integration

### Dynamic Tool Discovery

MCP servers are queried for available tools during registration and periodically for updates. The discovery process involves:

1. **Initial Discovery**: When an MCP server is registered, the system queries for all available tools
2. **Schema Validation**: Each tool's schema is validated against OpenAI strict mode requirements
3. **Health Assessment**: Tools are assessed for compatibility and marked with health status
4. **Integration**: Valid tools are integrated into the agent's tool registry

### Tool Metadata Management

Each discovered tool maintains comprehensive metadata including:

- **Schema Information**: JSON schema for parameter validation
- **Health Status**: Compliance with OpenAI strict mode requirements
- **Server Association**: Link to the originating MCP server
- **Execution Context**: Environment variables and agent-specific configurations

```mermaid
classDiagram
class MCPTool {
+string name
+string description
+dict inputSchema
+MCPToolHealth health
+validateSchema() bool
+normalizeSchema() dict
}
class MCPToolHealth {
+string status
+list reasons
+bool isValid()
}
class Tool {
+string name
+string description
+dict json_schema
+list tags
+dict metadata_
+ToolType tool_type
}
class MCPServer {
+string server_name
+MCPServerType server_type
+string server_url
+Secret token
+Dict custom_headers
+list tools
}
MCPTool --> MCPToolHealth : has
Tool --> MCPServer : belongs_to
MCPServer --> MCPTool : contains
```

**Diagram sources**
- [mcp.py](file://letta/schemas/mcp.py#L26-L147)
- [schema_validator.py](file://letta/functions/schema_validator.py#L12-L19)

**Section sources**
- [mcp_manager.py](file://letta/services/mcp_manager.py#L77-L100)
- [mcp_server_manager.py](file://letta/services/mcp_server_manager.py#L233-L273)

## Tool Execution Flow

### Execution Pipeline

The tool execution process follows a structured pipeline that ensures proper validation, authentication, and result processing:

```mermaid
sequenceDiagram
participant Agent as Agent
participant Executor as MCP Tool Executor
participant Manager as MCP Manager
participant Client as MCP Client
participant Server as MCP Server
Agent->>Executor : execute(function_name, args, tool)
Executor->>Executor : extract server name from tags
Executor->>Manager : execute_mcp_server_tool()
Manager->>Manager : validate tool existence
Manager->>Client : get_mcp_client()
Client->>Server : connect_to_server()
Server-->>Client : connection established
Client->>Server : call_tool(name, args)
Server-->>Client : tool result
Client-->>Manager : execution result
Manager-->>Executor : ToolExecutionResult
Executor-->>Agent : formatted result
```

**Diagram sources**
- [mcp_tool_executor.py](file://letta/services/tool_executor/mcp_tool_executor.py#L18-L56)
- [mcp_manager.py](file://letta/services/mcp_manager.py#L344-L371)

### Agent Integration

Agents integrate with MCP tools through a tagging system that associates tools with specific MCP servers:

1. **Tag Extraction**: Tools are identified by their MCP server tags
2. **Context Injection**: Agent-specific environment variables are injected
3. **Execution Context**: Agent state and sandbox configurations are maintained
4. **Result Processing**: Tool execution results are formatted for agent consumption

**Section sources**
- [mcp_tool_executor.py](file://letta/services/tool_executor/mcp_tool_executor.py#L18-L56)
- [mcp_manager.py](file://letta/services/mcp_manager.py#L344-L371)

## Schema Generation and Validation

### Schema Validation Process

The system implements comprehensive schema validation to ensure compatibility with OpenAI's strict mode requirements:

```mermaid
flowchart TD
Start([Tool Schema]) --> ValidateRoot{Root type object?}
ValidateRoot --> |No| Invalid[Mark INVALID]
ValidateRoot --> |Yes| CheckProps[Check Properties]
CheckProps --> PropsValid{Valid properties?}
PropsValid --> |No| Invalid
PropsValid --> |Yes| CheckRequired[Check Required Fields]
CheckRequired --> RequiredValid{Required fields valid?}
RequiredValid --> |No| Invalid
RequiredValid --> |Yes| CheckAdditional[Check additionalProperties]
CheckAdditional --> AdditionalValid{additionalProperties=false?}
AdditionalValid --> |No| NonStrict[Mark NON_STRICT_ONLY]
AdditionalValid --> |Yes| CheckTypes[Check Type Constraints]
CheckTypes --> TypesValid{Types valid?}
TypesValid --> |No| Invalid
TypesValid --> |Yes| StrictCompliant[Mark STRICT_COMPLIANT]
Invalid --> Return[Return INVALID status]
NonStrict --> Return
StrictCompliant --> Return
```

**Diagram sources**
- [schema_validator.py](file://letta/functions/schema_validator.py#L20-L203)

### Schema Normalization

The system automatically normalizes schemas to improve compatibility:

- **Type Enhancement**: Adds explicit types for `$ref` properties
- **Property Normalization**: Ensures `additionalProperties: false` for objects
- **Null Handling**: Converts optional fields to nullable types in strict mode
- **Duplicate Removal**: Eliminates duplicate entries in `anyOf` arrays

### Health Status Classification

| Status | Description | Action Required |
|--------|-------------|-----------------|
| `STRICT_COMPLIANT` | Fully compatible with OpenAI strict mode | No action needed |
| `NON_STRICT_ONLY` | Valid but not strict-compliant | May work but not optimal |
| `INVALID` | Broken schema requiring manual intervention | Manual correction needed |

**Section sources**
- [schema_validator.py](file://letta/functions/schema_validator.py#L12-L203)
- [mcp_manager.py](file://letta/services/mcp_manager.py#L147-L187)

## Authentication Mechanisms

### OAuth 2.0 Implementation

The system provides comprehensive OAuth 2.0 support for secure MCP server authentication:

```mermaid
sequenceDiagram
participant Client as Letta Client
participant Server as MCP Server
participant Auth as OAuth Provider
participant DB as Database
Client->>DB : Create OAuth session
Client->>Auth : Request authorization URL
Auth-->>Client : Authorization URL
Client->>Server : Redirect user to authorization URL
Server->>Server : User authenticates
Server->>Client : Callback with authorization code
Client->>Auth : Exchange code for tokens
Auth-->>Client : Access token + refresh token
Client->>DB : Store encrypted tokens
Client->>Server : Connect with OAuth
Server-->>Client : Authenticated connection
```

**Diagram sources**
- [oauth_utils.py](file://letta/services/mcp/oauth_utils.py#L192-L253)
- [mcp_servers.py](file://letta/server/rest_api/routers/v1/mcp_servers.py#L249-L296)

### API Token Authentication

For simpler authentication scenarios, the system supports API token authentication:

- **Bearer Token**: Standard OAuth Bearer token format
- **Custom Headers**: Configurable authentication headers
- **Environment Variables**: Support for environment-based credential injection

### Security Features

- **Encryption**: All sensitive credentials are encrypted at rest
- **Token Rotation**: Automatic refresh token handling
- **Secure Storage**: Database-backed token storage with access controls
- **Audit Logging**: Comprehensive logging of authentication events

**Section sources**
- [oauth_utils.py](file://letta/services/mcp/oauth_utils.py#L26-L93)
- [mcp.py](file://letta/schemas/mcp.py#L31-L66)

## Performance Considerations

### Network Latency Optimization

To minimize the impact of network latency on agent performance:

1. **Connection Pooling**: Reuse MCP client connections across tool executions
2. **Async Operations**: All MCP operations are performed asynchronously
3. **Timeout Management**: Configurable timeouts prevent hanging operations
4. **Circuit Breaker**: Automatic fallback for unresponsive servers

### Caching Strategies

The system implements multiple caching layers:

- **Tool Metadata Cache**: Persisted tool schemas to avoid repeated validation
- **Client Connection Cache**: Maintain active connections to reduce handshake overhead
- **Schema Cache**: Cache generated schemas to speed up subsequent operations

### Monitoring and Metrics

Key performance indicators tracked include:

- **Connection Success Rate**: Percentage of successful MCP server connections
- **Tool Discovery Time**: Time taken to discover tools from servers
- **Execution Latency**: Average time for tool execution requests
- **Error Rates**: Frequency of various error types

**Section sources**
- [base_client.py](file://letta/services/mcp/base_client.py#L29-L53)
- [mcp_manager.py](file://letta/services/mcp_manager.py#L77-L100)

## Error Handling and Troubleshooting

### Common Issues and Solutions

#### Connection Timeouts

**Symptoms**: MCP server connections fail with timeout errors
**Causes**: Network connectivity issues, server overload, incorrect URLs
**Solutions**:
- Verify server accessibility and network connectivity
- Check firewall and proxy configurations
- Adjust timeout settings in server configuration

#### Schema Compatibility Issues

**Symptoms**: Tools marked as `INVALID` in health status
**Causes**: Non-compliant JSON schemas, missing required fields
**Solutions**:
- Review schema validation errors in logs
- Use schema normalization features
- Contact server administrator for schema updates

#### Authentication Failures

**Symptoms**: 401 Unauthorized errors during tool execution
**Causes**: Expired tokens, incorrect credentials, OAuth configuration issues
**Solutions**:
- Refresh OAuth tokens through the admin interface
- Verify token expiration and renewal mechanisms
- Check OAuth provider configuration

### Error Propagation

The system implements structured error handling:

```mermaid
flowchart TD
Error[Error Occurred] --> Classify{Error Type}
Classify --> |Connection| ConnError[ConnectionError]
Classify --> |Authentication| AuthError[AuthenticationError]
Classify --> |Validation| ValError[ValidationError]
Classify --> |Execution| ExecError[ExecutionError]
ConnError --> Retry{Retry Available?}
AuthError --> Refresh[Refresh Tokens]
ValError --> Log[Log Validation Errors]
ExecError --> Report[Report Execution Failure]
Retry --> |Yes| RetryOp[Retry Operation]
Retry --> |No| Fail[Mark as Failed]
Refresh --> Retry
Log --> Fix[Fix Schema]
Report --> Monitor[Monitor Performance]
```

**Diagram sources**
- [base_client.py](file://letta/services/mcp/base_client.py#L29-L53)
- [exceptions.py](file://letta/functions/mcp_client/exceptions.py#L1-L6)

### Debugging Tools

The system provides comprehensive debugging capabilities:

- **Verbose Logging**: Detailed logs for troubleshooting connection issues
- **Schema Inspection**: Tools to inspect and validate tool schemas
- **Connection Testing**: Built-in tools to test server connectivity
- **Performance Monitoring**: Metrics for identifying bottlenecks

**Section sources**
- [base_client.py](file://letta/services/mcp/base_client.py#L29-L53)
- [exceptions.py](file://letta/functions/mcp_client/exceptions.py#L1-L6)

## Best Practices

### Server Configuration

1. **Use Descriptive Names**: Choose meaningful server names for easy identification
2. **Secure Credentials**: Always use encrypted storage for sensitive credentials
3. **Environment Variables**: Leverage environment variables for deployment flexibility
4. **Health Monitoring**: Regularly monitor server health and tool availability

### Tool Management

1. **Schema Validation**: Ensure all tools have valid JSON schemas
2. **Documentation**: Provide clear descriptions for all tools
3. **Testing**: Thoroughly test tools before deployment
4. **Version Control**: Track tool schema changes and updates

### Security Considerations

1. **Principle of Least Privilege**: Grant minimal necessary permissions
2. **Regular Audits**: Periodically review access permissions and credentials
3. **Monitoring**: Implement comprehensive logging and alerting
4. **Backup**: Maintain backups of server configurations and credentials

### Performance Optimization

1. **Connection Reuse**: Maintain persistent connections when possible
2. **Batch Operations**: Group related operations to reduce overhead
3. **Caching**: Implement appropriate caching strategies
4. **Monitoring**: Track performance metrics and optimize accordingly