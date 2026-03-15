
**Tags:** #grpc #graphql #protobuf #http2 #api #protocols #data-engineering

## gRPC Overview

gRPC (gRPC Remote Procedure Calls) is a high-performance, open-source RPC framework originally developed by Google. It uses **Protocol Buffers (Protobuf)** as its interface definition language and serialisation format, and runs over **HTTP/2** for transport.

### Core Components

| Component | Description |
|---|---|
| Protocol Buffers | Language-neutral, binary serialisation format for defining service contracts and message types |
| HTTP/2 Transport | Multiplexed connections, header compression, and full-duplex streaming over a single TCP connection |
| Binary Serialisation | Messages are encoded in a compact binary format — significantly smaller and faster than JSON or XML |
| Code Generation | `.proto` files are compiled into client and server stubs in any supported language |

### How gRPC Works

1. Define services and messages in a `.proto` file
2. Use the `protoc` compiler to generate client and server code in your target language
3. Implement the server-side logic behind the generated interface
4. Call remote methods from the client as if they were local function calls

> [!info] Language Support
> gRPC supports Python, Go, Java, C++, C#, Node.js, Ruby, Rust, and more. The generated code handles serialisation, deserialisation, and network transport automatically.

---

## gRPC Service Definition

### The `.proto` File

The `.proto` file is the single source of truth for a gRPC service. It defines both the **service interface** (methods) and the **message types** (data structures). This aligns closely with [[Data Contracts & Schema Enforcement]] principles — the schema is explicit, versioned, and enforced at compile time.

### Streaming Types

gRPC supports four communication patterns:

| Pattern | Client | Server | Use Case |
|---|---|---|---|
| **Unary** | Single request | Single response | Simple request/response (like REST) |
| **Server Streaming** | Single request | Stream of responses | Large result sets, real-time feeds |
| **Client Streaming** | Stream of requests | Single response | File uploads, batch ingestion |
| **Bidirectional Streaming** | Stream of requests | Stream of responses | Chat, real-time telemetry, event processing |

Server and bidirectional streaming patterns are particularly relevant for data engineering workloads where continuous data flow is needed — similar to how [[Apache Kafka Fundamentals]] handles event streams, but at the RPC level rather than the message broker level.

---

## gRPC vs REST Comparison

| Dimension | gRPC | [[REST APIs\|REST]] |
|---|---|---|
| **Serialisation** | Binary (Protobuf) — compact, fast | Text (JSON/XML) — human-readable |
| **Transport** | HTTP/2 (multiplexed, bidirectional) | HTTP/1.1 or HTTP/2 |
| **Schema** | Strongly typed `.proto` contract | Optional (OpenAPI/Swagger) |
| **Streaming** | Native support (all four patterns) | Limited (SSE, WebSockets are separate) |
| **Performance** | 2-10x faster serialisation, smaller payloads | Adequate for most workloads |
| **Browser Support** | Limited (requires gRPC-Web proxy) | Universal |
| **Tooling** | Specialised (BloomRPC, grpcurl, Evans) | Extensive (Postman, curl, browser) |
| **Human Readability** | Binary — not human-readable on the wire | JSON is easy to inspect and debug |
| **Code Generation** | Built-in from `.proto` files | Optional (OpenAPI codegen) |
| **Versioning** | Field numbering with backward compatibility | URL versioning (`/v1/`, `/v2/`) |

---

## When To Use gRPC In Data Engineering

gRPC excels in specific data engineering scenarios:

- **Microservice-to-Microservice Communication** — where services are internal, latency matters, and both sides can use generated clients. Particularly effective in polyglot environments where services are written in different languages.
- **ML Model Serving** — frameworks like TensorFlow Serving and Seldon Core use gRPC for low-latency inference requests. The binary serialisation handles large numeric tensors efficiently.
- **High-Throughput Data Pipelines** — streaming RPCs for continuous ingestion where [[Apache Kafka Fundamentals|Kafka]] would add unnecessary infrastructure overhead for point-to-point communication.
- **Inter-Platform Communication** — connecting Spark drivers to external services, or orchestrating distributed compute tasks where type safety prevents runtime errors.
- **Real-Time Feature Stores** — serving precomputed features to ML models with sub-millisecond latency requirements.

> [!warning] When Not To Use gRPC
> Avoid gRPC for public-facing APIs, browser clients without a proxy layer, or when human readability of payloads is a priority. For most external integrations, [[REST APIs]] remain the better choice.

---

## GraphQL Overview

GraphQL is a query language for APIs and a runtime for executing those queries against your data. Developed by Facebook in 2012 and open-sourced in 2015, it follows a **schema-first design** philosophy where the API's capabilities are defined by a strongly typed schema.

### Core Concepts

| Concept | Description |
|---|---|
| **Schema** | The complete type system that defines all available data and operations |
| **Queries** | Read operations — clients specify exactly which fields they need |
| **Mutations** | Write operations — create, update, or delete data |
| **Subscriptions** | Real-time updates pushed to clients over WebSockets |
| **Resolvers** | Server-side functions that fetch the data for each field in the schema |

### Key Principles

- **Declarative Data Fetching** — the client describes what data it needs, not how to get it
- **Single Endpoint** — all operations go through one URL (typically `/graphql`)
- **Hierarchical** — queries mirror the shape of the response, making them intuitive to write
- **Strongly Typed** — every field has a defined type; the schema is the contract

---

## GraphQL Type System

GraphQL's type system is the foundation of every API. It defines the shape of available data and the operations clients can perform.

### Built-In Scalar Types

| Type | Description | Example |
|---|---|---|
| `Int` | Signed 32-bit integer | `42` |
| `Float` | Double-precision floating point | `3.14` |
| `String` | UTF-8 character sequence | `"hello"` |
| `Boolean` | True or false | `true` |
| `ID` | Unique identifier (serialised as String) | `"abc-123"` |

### Object Types, Fields, and Arguments

```graphql
type Pipeline {
  id: ID!
  name: String!
  status: PipelineStatus!
  lastRunAt: String
  tasks(limit: Int = 10): [Task!]!
}

enum PipelineStatus {
  RUNNING
  SUCCEEDED
  FAILED
  PENDING
}

type Task {
  id: ID!
  name: String!
  duration: Float
  logs: [String!]
}
```

- The `!` suffix denotes a non-nullable field
- Arguments (like `limit` on `tasks`) allow field-level parameterisation
- Enums constrain values to a defined set

### Resolvers

Resolvers are functions that populate each field in the schema. A resolver for the `tasks` field on `Pipeline` might query a database, call a microservice, or read from a cache. This decoupling means a single GraphQL schema can aggregate data from multiple backends.

---

## GraphQL vs REST Comparison

| Dimension | GraphQL | [[REST APIs\|REST]] |
|---|---|---|
| **Over-Fetching** | Eliminated — client selects exact fields | Common — endpoints return fixed payloads |
| **Under-Fetching** | Eliminated — nested data in one query | Common — requires multiple round trips |
| **Endpoints** | Single endpoint (`/graphql`) | Multiple endpoints per resource |
| **Versioning** | Typically not needed (additive schema evolution) | URL or header versioning required |
| **Caching** | More complex (POST-based, needs persisted queries) | Simple (HTTP caching, ETags, CDN-friendly) |
| **Error Handling** | Always returns 200; errors in response body | Uses HTTP status codes (4xx, 5xx) |
| **Discoverability** | Introspection queries expose full schema | Requires external docs (Swagger/OpenAPI) |
| **File Uploads** | Not natively supported (multipart workarounds) | Straightforward with multipart/form-data |
| **Tooling** | GraphiQL, Apollo Studio, Hasura | Postman, curl, Swagger UI |
| **Learning Curve** | Steeper — new query language and concepts | Lower — leverages existing HTTP knowledge |

---

## When To Use GraphQL In Data Engineering

GraphQL fits particular data engineering patterns:

- **API Aggregation Layer** — when a single gateway must unify data from multiple microservices, databases, or third-party APIs into a coherent interface. GraphQL's resolver architecture is purpose-built for this.
- **Flexible Data Access For Analysts** — data catalogues and metadata platforms (e.g., DataHub, Amundsen) use GraphQL to let users explore schemas, lineage, and quality metrics without rigid endpoint structures.
- **Self-Service Data Products** — teams consuming data products can query exactly the fields they need without waiting for backend changes — aligning with data mesh principles.
- **Dashboard and Reporting Backends** — where different dashboards need different slices of the same data, eliminating the need for dozens of bespoke REST endpoints.

> [!warning] When Not To Use GraphQL
> Avoid GraphQL for simple CRUD APIs, server-to-server batch processing, file-heavy operations, or when HTTP caching is critical. The overhead of a query parser and resolver chain is unnecessary for straightforward pipeline integrations.

---

## Code Examples

### Protobuf Service Definition (`.proto`)

```protobuf
syntax = "proto3";

package dataplatform;

// Service for managing data pipeline executions
service PipelineService {
  // Unary: trigger a single pipeline run
  rpc TriggerRun (TriggerRequest) returns (RunResponse);

  // Server streaming: follow run logs in real time
  rpc StreamLogs (LogRequest) returns (stream LogEntry);

  // Client streaming: ingest a batch of records
  rpc IngestRecords (stream Record) returns (IngestSummary);
}

message TriggerRequest {
  string pipeline_id = 1;
  map<string, string> parameters = 2;
}

message RunResponse {
  string run_id = 1;
  string status = 2;
  string started_at = 3;
}

message LogRequest {
  string run_id = 1;
}

message LogEntry {
  string timestamp = 1;
  string level = 2;
  string message = 3;
}

message Record {
  string key = 1;
  bytes payload = 2;
}

message IngestSummary {
  int64 records_received = 1;
  int64 records_accepted = 2;
  int64 records_rejected = 3;
}
```

### gRPC Python Client

```python
import grpc
from dataplatform import pipeline_pb2, pipeline_pb2_grpc

def trigger_pipeline(pipeline_id: str, params: dict[str, str]) -> str:
    """Trigger a pipeline run and stream its logs."""
    channel = grpc.insecure_channel("localhost:50051")
    stub = pipeline_pb2_grpc.PipelineServiceStub(channel)

    # Unary call — trigger the run
    request = pipeline_pb2.TriggerRequest(
        pipeline_id=pipeline_id,
        parameters=params,
    )
    response = stub.TriggerRun(request)
    print(f"Run started: {response.run_id} — status: {response.status}")

    # Server streaming — follow logs
    log_request = pipeline_pb2.LogRequest(run_id=response.run_id)
    for log in stub.StreamLogs(log_request):
        print(f"[{log.timestamp}] {log.level}: {log.message}")

    return response.run_id


if __name__ == "__main__":
    trigger_pipeline("etl-daily-sales", {"date": "2026-03-15"})
```

### GraphQL Schema

```graphql
type Query {
  pipeline(id: ID!): Pipeline
  pipelines(status: PipelineStatus, limit: Int = 20): [Pipeline!]!
  dataset(name: String!): Dataset
}

type Mutation {
  triggerPipeline(id: ID!, parameters: JSON): PipelineRun!
  updateSchedule(pipelineId: ID!, cron: String!): Pipeline!
}

type Subscription {
  pipelineStatusChanged(id: ID!): PipelineRun!
}

type Pipeline {
  id: ID!
  name: String!
  description: String
  schedule: String
  status: PipelineStatus!
  lastRun: PipelineRun
  datasets: [Dataset!]!
}

type PipelineRun {
  id: ID!
  status: PipelineStatus!
  startedAt: String!
  completedAt: String
  recordsProcessed: Int
}

type Dataset {
  name: String!
  schema: [Column!]!
  rowCount: Int
  lastUpdated: String
}

type Column {
  name: String!
  dataType: String!
  nullable: Boolean!
}

enum PipelineStatus {
  RUNNING
  SUCCEEDED
  FAILED
  PENDING
  SCHEDULED
}

scalar JSON
```

### GraphQL Query

```graphql
# Fetch a pipeline with its recent run and output datasets
query GetPipelineDetails {
  pipeline(id: "etl-daily-sales") {
    name
    status
    schedule
    lastRun {
      status
      startedAt
      completedAt
      recordsProcessed
    }
    datasets {
      name
      rowCount
      lastUpdated
      schema {
        name
        dataType
        nullable
      }
    }
  }
}
```

**Response:**

```json
{
  "data": {
    "pipeline": {
      "name": "ETL Daily Sales",
      "status": "SUCCEEDED",
      "schedule": "0 6 * * *",
      "lastRun": {
        "status": "SUCCEEDED",
        "startedAt": "2026-03-15T06:00:00Z",
        "completedAt": "2026-03-15T06:12:34Z",
        "recordsProcessed": 284710
      },
      "datasets": [
        {
          "name": "fact_sales",
          "rowCount": 14230500,
          "lastUpdated": "2026-03-15T06:12:34Z",
          "schema": [
            { "name": "sale_id", "dataType": "BIGINT", "nullable": false },
            { "name": "amount", "dataType": "DECIMAL(12,2)", "nullable": false },
            { "name": "sale_date", "dataType": "DATE", "nullable": false }
          ]
        }
      ]
    }
  }
}
```

---

## Protocol Decision Matrix

Neither gRPC nor GraphQL replaces [[REST APIs]] for most data pipelines. Each protocol has a sweet spot, and the right choice depends on the specific integration pattern.

| Criterion | REST | gRPC | GraphQL |
|---|---|---|---|
| **Public-Facing API** | Best choice | Poor (limited browser support) | Good (flexible queries) |
| **Microservice Comms** | Adequate | Best choice | Overhead not justified |
| **ML Model Serving** | Adequate | Best choice | Not applicable |
| **Data Catalogue / Metadata** | Adequate | Not ideal | Best choice |
| **Batch Pipeline Integration** | Good | Unnecessary complexity | Unnecessary complexity |
| **Real-Time Streaming** | Limited | Excellent (native streaming) | Limited (subscriptions only) |
| **Mobile / Frontend** | Good | Requires proxy | Good (reduces round trips) |
| **Schema Enforcement** | Optional | Enforced (Protobuf) | Enforced (type system) |
| **Human Debugging** | Easy (JSON + curl) | Difficult (binary) | Moderate (single endpoint) |
| **Infrastructure Maturity** | Universal | Moderate | Moderate |

### Decision Guidelines

1. **Default to REST** for external APIs, simple integrations, and pipeline orchestration webhooks
2. **Choose gRPC** when you need high throughput, low latency, streaming, or type-safe inter-service communication — especially in polyglot microservice architectures
3. **Choose GraphQL** when consumers need flexible, self-service access to complex data — data catalogues, metadata APIs, or frontend-driven analytics
4. **Combine protocols** in practice — many platforms expose REST externally, use gRPC internally between services, and offer GraphQL for developer portals

> [!tip] In Practice
> Most mature data platforms use all three protocols in different layers. [[REST APIs]] handle external integrations, gRPC handles internal service mesh communication, and GraphQL serves the metadata and catalogue layer. The key is choosing the right tool for each boundary.

---

## Related Topics

- [[REST APIs]] — the dominant API paradigm for data pipeline integrations
- [[SOAP (Simple Object Access Protocol)|SOAP]] — legacy protocol comparison (XML-based, strict contracts)
- [[Data Contracts & Schema Enforcement]] — schema-first design principles shared by gRPC and GraphQL
- [[Apache Kafka Fundamentals]] — event streaming as an alternative to RPC-based communication
