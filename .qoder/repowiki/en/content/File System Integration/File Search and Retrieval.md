# File Search and Retrieval

<cite>
**Referenced Files in This Document**
- [passage_manager.py](file://letta/services/passage_manager.py)
- [file_processor.py](file://letta/services/file_processor/file_processor.py)
- [base_embedder.py](file://letta/services/file_processor/embedder/base_embedder.py)
- [llama_index_chunker.py](file://letta/services/file_processor/chunker/llama_index_chunker.py)
- [files_tool_executor.py](file://letta/services/tool_executor/files_tool_executor.py)
- [tpuf_client.py](file://letta/helpers/tpuf_client.py)
- [pinecone_embedder.py](file://letta/services/file_processor/embedder/pinecone_embedder.py)
- [turbopuffer_embedder.py](file://letta/services/file_processor/embedder/turbopuffer_embedder.py)
- [passage.py](file://letta/schemas/passage.py)
- [file_types.py](file://letta/services/file_processor/file_types.py)
- [agent_manager_helper.py](file://letta/services/helpers/agent_manager_helper.py)
- [openai_client.py](file://letta/llm_api/openai_client.py)
- [source_manager.py](file://letta/services/source_manager.py)
- [agent_manager.py](file://letta/services/agent_manager.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [System Architecture](#system-architecture)
3. [Passage Management](#passage-management)
4. [File Processing Pipeline](#file-processing-pipeline)
5. [Vector Database Integration](#vector-database-integration)
6. [Search Capabilities](#search-capabilities)
7. [Agent Tools for File Search](#agent-tools-for-file-search)
8. [Performance Optimization](#performance-optimization)
9. [Troubleshooting Guide](#troubleshooting-guide)
10. [Best Practices](#best-practices)

## Introduction

Letta's file search and retrieval system provides sophisticated capabilities for querying document contents across large collections of files. The system combines semantic search with traditional text search to deliver relevant results with high precision. It supports multiple vector database providers, intelligent chunking strategies, and comprehensive metadata filtering.

The core functionality revolves around the PassageManager service, which handles the creation, storage, and retrieval of document passages. These passages are embedded representations of text chunks that enable semantic similarity search across millions of documents.

## System Architecture

The file search and retrieval system follows a layered architecture that separates concerns between file processing, passage management, and search execution.

```mermaid
graph TB
subgraph "File Processing Layer"
FM[File Manager]
FP[File Processor]
Parser[File Parser]
Chunker[LlamaIndex Chunker]
Embedder[Embedder]
end
subgraph "Passage Management Layer"
PM[Passage Manager]
PassageStore[Passage Storage]
TagManager[Tag Manager]
end
subgraph "Search Layer"
SearchEngine[Search Engine]
VectorDB[Vector Database]
MetadataFilter[Metadata Filter]
end
subgraph "Agent Tools"
FileTools[File Search Tools]
SemanticSearch[Semantic Search]
GrepFiles[Grep Files]
end
FM --> FP
FP --> Parser
FP --> Chunker
FP --> Embedder
Embedder --> PM
PM --> PassageStore
PM --> TagManager
SearchEngine --> VectorDB
SearchEngine --> MetadataFilter
PM --> SearchEngine
FileTools --> SearchEngine
SemanticSearch --> SearchEngine
GrepFiles --> SearchEngine
```

**Diagram sources**
- [file_processor.py](file://letta/services/file_processor/file_processor.py#L27-L48)
- [passage_manager.py](file://letta/services/passage_manager.py#L51-L56)
- [files_tool_executor.py](file://letta/services/tool_executor/files_tool_executor.py#L308-L728)

**Section sources**
- [file_processor.py](file://letta/services/file_processor/file_processor.py#L24-L139)
- [passage_manager.py](file://letta/services/passage_manager.py#L51-L56)

## Passage Management

The PassageManager service serves as the central hub for all passage-related operations, handling both archival passages (agent memory) and source passages (file content).

### Core Passage Types

The system distinguishes between two primary passage types:

**Archival Passages**: Stored in agent archives for long-term memory retention. These passages are managed by the PassageManager and support dual storage in both SQL databases and vector databases.

**Source Passages**: Created during file processing and stored in vector databases for search purposes. These passages maintain associations with their source files and metadata.

### Passage Creation Workflow

```mermaid
sequenceDiagram
participant FP as File Processor
participant E as Embedder
participant PM as Passage Manager
participant VDB as Vector Database
participant SQL as SQL Database
FP->>E : Generate embeddings
E->>E : Process text chunks
E->>PM : Create passages
PM->>SQL : Store passage metadata
PM->>VDB : Store embeddings
PM->>PM : Create tag junction records
PM-->>FP : Return passages
```

**Diagram sources**
- [file_processor.py](file://letta/services/file_processor/file_processor.py#L49-L139)
- [passage_manager.py](file://letta/services/passage_manager.py#L141-L251)

### Passage Schema and Features

Passages contain rich metadata and support advanced features:

| Field | Type | Description | Purpose |
|-------|------|-------------|---------|
| id | String | Unique passage identifier | Primary key for retrieval |
| text | String | Original passage text | Content storage |
| embedding | Array[float] | Vector representation | Semantic search |
| embedding_config | EmbeddingConfig | Embedding parameters | Configuration reference |
| organization_id | String | Organization association | Multi-tenancy |
| source_id | String | Source file reference | File provenance |
| file_id | String | File association | File tracking |
| tags | Array[string] | Classification tags | Filtering and categorization |
| metadata | Object | Additional metadata | Extended attributes |

**Section sources**
- [passage.py](file://letta/schemas/passage.py#L14-L95)
- [passage_manager.py](file://letta/services/passage_manager.py#L141-L251)

## File Processing Pipeline

The file processing pipeline transforms raw documents into searchable passages through a sophisticated multi-stage process.

### Processing Stages

```mermaid
flowchart TD
Start([File Upload]) --> DetectType[Detect File Type]
DetectType --> ParseContent[Parse Content]
ParseContent --> ChunkText[Apply Chunking Strategy]
ChunkText --> GenerateEmbeddings[Generate Embeddings]
GenerateEmbeddings --> CreatePassages[Create Passage Objects]
CreatePassages --> StoreMetadata[Store Metadata]
StoreMetadata --> StoreVectors[Store Vectors]
StoreVectors --> CreateTags[Create Tag Junctions]
CreateTags --> Complete([Processing Complete])
ParseContent --> ParseError{Parse Error?}
ParseError --> |Yes| FallbackParse[Fallback Parsing]
FallbackParse --> ChunkText
ChunkText --> ChunkError{Chunk Error?}
ChunkError --> |Yes| FallbackChunk[Fallback Chunking]
FallbackChunk --> GenerateEmbeddings
```

**Diagram sources**
- [file_processor.py](file://letta/services/file_processor/file_processor.py#L49-L139)
- [llama_index_chunker.py](file://letta/services/file_processor/chunker/llama_index_chunker.py#L88-L170)

### File Type Detection and Chunking Strategies

The system supports intelligent chunking based on file type characteristics:

| File Type Category | Chunking Strategy | Description | Use Case |
|-------------------|------------------|-------------|----------|
| Code Files | CODE | Line-based with syntax awareness | Source code analysis |
| Documentation | DOCUMENTATION | Paragraph-aware with structure preservation | Markdown, HTML, documentation |
| Structured Data | STRUCTURED_DATA | JSON/XML node parsing | Configuration files, APIs |
| Prose Documents | LINE_BASED | Sentence-aware splitting | Articles, reports |

### Embedding Generation and Batching

The embedding system employs sophisticated batching and retry mechanisms:

```mermaid
sequenceDiagram
participant FP as File Processor
participant EC as Embedding Client
participant API as Embedding API
participant Cache as Embedding Cache
FP->>EC : Request embeddings
EC->>Cache : Check cache
Cache-->>EC : Cache miss
EC->>EC : Batch inputs
EC->>API : Send batch request
API-->>EC : Return embeddings
EC->>Cache : Store results
EC-->>FP : Return processed embeddings
```

**Diagram sources**
- [openai_client.py](file://letta/llm_api/openai_client.py#L744-L824)
- [base_embedder.py](file://letta/services/file_processor/embedder/base_embedder.py#L19-L22)

**Section sources**
- [file_processor.py](file://letta/services/file_processor/file_processor.py#L49-L139)
- [llama_index_chunker.py](file://letta/services/file_processor/chunker/llama_index_chunker.py#L12-L170)
- [base_embedder.py](file://letta/services/file_processor/embedder/base_embedder.py#L12-L22)

## Vector Database Integration

Letta supports multiple vector database providers, each optimized for different use cases and scale requirements.

### Supported Providers

```mermaid
graph LR
subgraph "Vector Database Options"
Native[Native SQLite<br/>Development/Small Scale]
TPuf[Turbopuffer<br/>Cloud-native<br/>Production Ready]
Pinecone[Pinecone<br/>Managed Service<br/>Enterprise Scale]
end
subgraph "Features"
Native --> NativeFeatures[Local storage,<br/>No external deps]
TPuf --> TPufFeatures[Cloud-native,<br/>Auto-scaling,<br/>Dual-write support]
Pinecone --> PineconeFeatures[Managed infrastructure,<br/>High availability,<br/>Advanced analytics]
end
```

**Diagram sources**
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L22-L30)
- [pinecone_embedder.py](file://letta/services/file_processor/embedder/pinecone_embedder.py#L20-L35)
- [turbopuffer_embedder.py](file://letta/services/file_processor/embedder/turbopuffer_embedder.py#L16-L34)

### Provider Configuration and Selection

The system automatically selects the appropriate provider based on configuration and availability:

| Provider | Configuration | Use Case | Performance |
|----------|---------------|----------|-------------|
| TPUF | `use_tpuf=true` + API key | Production deployments | High throughput, auto-scaling |
| PINECONE | Pinecone credentials | Enterprise environments | Managed scalability |
| NATIVE | Default SQLite | Development/testing | Local performance |

### Dual-Write Architecture

For production deployments, the system supports dual-write to both SQL and vector databases:

```mermaid
sequenceDiagram
participant Agent as Agent
participant PM as Passage Manager
participant SQL as SQL Database
participant VDB as Vector Database
participant TPuf as Turbopuffer
Agent->>PM : Create passage
PM->>SQL : Store metadata
PM->>VDB : Store embeddings
PM->>TPuf : Write to Turbopuffer
TPuf-->>PM : Confirm write
PM-->>Agent : Return passage
```

**Diagram sources**
- [passage_manager.py](file://letta/services/passage_manager.py#L517-L542)
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L201-L225)

**Section sources**
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L22-L30)
- [pinecone_embedder.py](file://letta/services/file_processor/embedder/pinecone_embedder.py#L20-L35)
- [turbopuffer_embedder.py](file://letta/services/file_processor/embedder/turbopuffer_embedder.py#L16-L34)

## Search Capabilities

The search system provides multiple search modes and sophisticated filtering capabilities.

### Search Modes

```mermaid
graph TB
subgraph "Search Modes"
Vector[Vector Search<br/>Semantic Similarity]
FTS[Full-Text Search<br/>Keyword Matching]
Hybrid[Hybrid Search<br/>Combined Results]
Timestamp[Timestamp Search<br/>Chronological]
end
subgraph "Scoring Methods"
Cosine[Cosine Distance<br/>Vector Similarity]
BM25[BM25 Score<br/>Text Relevance]
RRF[Reciprocal Rank Fusion<br/>Hybrid Ranking]
end
Vector --> Cosine
FTS --> BM25
Hybrid --> RRF
```

**Diagram sources**
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L421-L442)
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L573-L590)

### Semantic Search Implementation

Semantic search uses vector similarity to find conceptually related content:

```mermaid
sequenceDiagram
participant Agent as Agent
participant Search as Search Engine
participant Embedder as Embedding Generator
participant VectorDB as Vector Database
participant Scorer as Result Scorer
Agent->>Search : Query text
Search->>Embedder : Generate query embedding
Embedder-->>Search : Query vector
Search->>VectorDB : Vector similarity search
VectorDB-->>Search : Top-K similar passages
Search->>Scorer : Calculate relevance scores
Scorer-->>Search : Ranked results
Search-->>Agent : Formatted results
```

**Diagram sources**
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L486-L506)
- [agent_manager_helper.py](file://letta/services/helpers/agent_manager_helper.py#L985-L992)

### Metadata Filtering and Tagging

The system supports comprehensive filtering using metadata and tags:

| Filter Type | Description | Example Usage |
|-------------|-------------|---------------|
| Source ID | Filter by source origin | `source_id: "source_123"` |
| File ID | Filter by specific file | `file_id: "file_456"` |
| Date Range | Temporal filtering | `created_at >= "2024-01-01"` |
| Tags | Content classification | `tags: ["important", "finance"]` |
| Organization | Multi-tenancy isolation | `organization_id: "org_789"` |

### Hybrid Search with Reciprocal Rank Fusion

For optimal results, the system combines vector and text search using Reciprocal Rank Fusion (RRF):

```mermaid
flowchart TD
Query[Search Query] --> VectorSearch[Vector Search]
Query --> FTSSearch[Full-Text Search]
VectorSearch --> VectorResults[Vector Results]
FTSSearch --> FTSResults[FTS Results]
VectorResults --> RRF[Reciprocal Rank Fusion]
FTSResults --> RRF
RRF --> CombinedScores[Combined Scores]
CombinedScores --> FinalRanking[Final Ranking]
FinalRanking --> RankedResults[Ranked Results]
```

**Diagram sources**
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L573-L589)

**Section sources**
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L421-L590)
- [agent_manager_helper.py](file://letta/services/helpers/agent_manager_helper.py#L962-L1091)

## Agent Tools for File Search

Agents can interact with the file search system through specialized tools that provide natural language interfaces to search capabilities.

### Available Search Tools

The file search tools provide comprehensive search functionality:

| Tool | Parameters | Description | Use Case |
|------|------------|-------------|----------|
| `grep_files` | `pattern`, `include`, `context_lines`, `offset` | Regex-based file search | Pattern matching across files |
| `semantic_search_files` | `query`, `limit` | Semantic similarity search | Finding conceptually related content |
| `search_files` | `query`, `limit`, `filters` | General file search | Flexible search with multiple criteria |

### Search Tool Parameters

```mermaid
classDiagram
class GrepFilesParams {
+string pattern
+string include
+int context_lines
+int offset
+validatePattern()
+compileRegex()
}
class SemanticSearchParams {
+string query
+int limit
+float vector_weight
+float fts_weight
+dict filters
}
class SearchParams {
+string query
+int limit
+dict filters
+string search_mode
+validateParameters()
}
GrepFilesParams --|> SearchParams
SemanticSearchParams --|> SearchParams
```

**Diagram sources**
- [files_tool_executor.py](file://letta/services/tool_executor/files_tool_executor.py#L311-L330)
- [files_tool_executor.py](file://letta/services/tool_executor/files_tool_executor.py#L539-L542)

### Search Examples

**Semantic Search Example**:
```python
# Agent searches for financial documents
results = await semantic_search_files(
    agent_state=agent,
    query="financial statements and revenue analysis",
    limit=5
)
```

**Pattern Search Example**:
```python
# Agent searches for specific patterns across files
results = await grep_files(
    agent_state=agent,
    pattern=r"ERROR \d{3}:.*",
    include="*.log",
    context_lines=2
)
```

**Section sources**
- [files_tool_executor.py](file://letta/services/tool_executor/files_tool_executor.py#L308-L728)

## Performance Optimization

The system implements several performance optimization strategies for handling large document collections efficiently.

### Indexing Strategies

```mermaid
graph TB
subgraph "Indexing Layers"
FileLevel[File-Level Indexing<br/>File metadata, structure]
PassageLevel[Passage-Level Indexing<br/>Text chunks, embeddings]
TagLevel[Tag-Level Indexing<br/>Classification, filtering]
end
subgraph "Optimization Techniques"
Batching[Batch Processing<br/>Concurrent embedding generation]
Caching[Result Caching<br/>Embedding cache, query results]
Parallel[Parallel Processing<br/>Async operations, thread pools]
end
FileLevel --> Batching
PassageLevel --> Caching
TagLevel --> Parallel
```

**Diagram sources**
- [file_processor.py](file://letta/services/file_processor/file_processor.py#L49-L139)
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L18-L20)

### Caching Mechanisms

The system employs multiple caching layers:

| Cache Level | Technology | Purpose | Duration |
|-------------|------------|---------|----------|
| Embedding Cache | Redis/LRU | Prevent redundant embeddings | Session-based |
| Query Cache | Application | Store search results | Configurable |
| File Content Cache | Memory | Reduce file I/O | Runtime |
| Vector Cache | Vector DB | Optimize similarity queries | Persistent |

### Concurrency and Throttling

```mermaid
sequenceDiagram
participant Agent as Agent
participant Semaphore as Global Semaphore
participant ThreadPool as Thread Pool
participant API as External API
Agent->>Semaphore : Acquire slot
Semaphore-->>Agent : Slot granted
Agent->>ThreadPool : Submit chunking task
ThreadPool->>API : Process chunk
API-->>ThreadPool : Return result
ThreadPool-->>Agent : Task complete
Agent->>Semaphore : Release slot
```

**Diagram sources**
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L18-L20)
- [openai_client.py](file://letta/llm_api/openai_client.py#L744-L824)

### Large Collection Handling

For large document collections, the system implements:

- **Streaming Processing**: Process files in chunks to manage memory usage
- **Progressive Loading**: Load file contents on-demand for agents
- **Lazy Evaluation**: Defer expensive operations until needed
- **Resource Limits**: Configurable limits on concurrent operations

**Section sources**
- [tpuf_client.py](file://letta/helpers/tpuf_client.py#L18-L20)
- [openai_client.py](file://letta/llm_api/openai_client.py#L744-L824)
- [file_processor.py](file://letta/services/file_processor/file_processor.py#L49-L139)

## Troubleshooting Guide

Common issues and their solutions when working with file search and retrieval.

### Embedding Quality Issues

**Problem**: Poor search relevance despite good content
**Symptoms**: 
- Irrelevant results for semantic queries
- Low similarity scores
- Inconsistent search behavior

**Solutions**:
1. **Verify Embedding Model**: Ensure the embedding model matches your content type
2. **Check Text Preprocessing**: Validate that text cleaning removes noise
3. **Review Chunk Size**: Adjust chunk size for optimal granularity
4. **Monitor Embedding Dimensions**: Verify embeddings match expected dimensions

### Search Relevance Problems

**Problem**: Search results don't match expectations
**Symptoms**:
- Missing relevant documents
- Too many irrelevant results
- Slow search performance

**Diagnostic Steps**:
```mermaid
flowchart TD
Issue[Search Issue] --> CheckQuery{Query Problem?}
CheckQuery --> |Yes| FixQuery[Refine Query<br/>Improve specificity<br/>Use tags/metadata]
CheckQuery --> |No| CheckIndex{Index Problem?}
CheckIndex --> |Yes| RebuildIndex[Rebuild Index<br/>Check embeddings<br/>Validate chunking]
CheckIndex --> |No| CheckConfig{Configuration?}
CheckConfig --> |Yes| FixConfig[Adjust settings<br/>Increase limits<br/>Enable caching]
CheckConfig --> |No| CheckData{Data Problem?}
CheckData --> |Yes| FixData[Clean data<br/>Fix encoding<br/>Validate structure]
CheckData --> |No| ContactSupport[Contact Support<br/>Check logs<br/>Report bug]
```

### Vector Database Issues

**Connection Problems**:
- Verify API keys and credentials
- Check network connectivity
- Validate database configuration

**Performance Issues**:
- Monitor embedding generation rates
- Check vector database capacity
- Review query complexity

**Data Integrity Issues**:
- Verify passage counts match expectations
- Check for duplicate embeddings
- Validate metadata consistency

### File Processing Failures

**Common Failure Points**:
1. **Unsupported File Types**: Verify file type registration
2. **Large File Handling**: Check memory limits and timeouts
3. **Encoding Issues**: Validate character encoding
4. **Parsing Errors**: Review fallback mechanisms

**Section sources**
- [openai_client.py](file://letta/llm_api/openai_client.py#L744-L824)
- [file_processor.py](file://letta/services/file_processor/file_processor.py#L49-L139)

## Best Practices

### File Organization and Naming

- Use descriptive file names that reflect content
- Organize files in logical directory structures
- Maintain consistent naming conventions
- Include version information in file names

### Embedding Configuration

- Choose embedding models appropriate for your content type
- Configure chunk sizes based on document nature
- Monitor embedding quality metrics
- Implement embedding validation checks

### Search Strategy

- Combine semantic and keyword search for best results
- Use tags for content classification and filtering
- Implement progressive refinement for complex queries
- Cache frequently accessed search results

### Performance Tuning

- Monitor embedding generation rates and optimize batch sizes
- Implement appropriate caching strategies
- Use vector database indexes effectively
- Configure resource limits based on workload

### Monitoring and Maintenance

- Track passage creation and search performance metrics
- Monitor embedding quality and relevance scores
- Regular index maintenance and optimization
- Implement alerting for system failures