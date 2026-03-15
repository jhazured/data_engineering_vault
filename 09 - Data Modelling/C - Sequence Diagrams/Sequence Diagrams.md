# Sequence Diagrams

Sequence diagrams show the order of interactions between components over time — essential for documenting APIs, pipelines, and multi-step processes.

## Core Elements

| Element | Symbol | Purpose |
|---------|--------|---------|
| **Participant** | Box at top | Actor or system component |
| **Lifeline** | Dashed vertical line | Time flowing downward |
| **Message** | Horizontal arrow | Interaction between participants |
| **Activation** | Narrow rectangle on lifeline | Period when participant is active |
| **Return** | Dashed arrow | Response to a message |
| **Note** | Rectangle with folded corner | Annotation |

## Data Pipeline Sequences

### Incremental dbt Pipeline

```mermaid
sequenceDiagram
    participant CI as GitLab CI
    participant dbt as dbt
    participant SF as Snowflake
    participant T0 as T0 Control

    CI->>dbt: dbt run --select tag:load_priority_1
    activate dbt
    dbt->>SF: SELECT MAX(_ingested_at) FROM T2 staging
    SF-->>dbt: watermark timestamp
    dbt->>SF: SELECT * FROM T1 WHERE _loaded_at > watermark
    SF-->>dbt: new rows
    dbt->>SF: MERGE INTO T2 staging
    SF-->>dbt: rows merged
    deactivate dbt

    CI->>dbt: dbt snapshot
    activate dbt
    dbt->>SF: Compare T2 rows with snapshot
    dbt->>SF: INSERT new SCD2 versions
    deactivate dbt

    CI->>dbt: dbt run --select tag:load_priority_2a
    activate dbt
    dbt->>SF: Build T3 dimensions from snapshots
    deactivate dbt

    CI->>dbt: dbt test --select tag:critical
    activate dbt
    dbt->>SF: Run data quality tests
    SF-->>dbt: pass/fail results
    dbt->>T0: INSERT INTO TBL_PIPELINE_RUN_LOG
    deactivate dbt
```

### RAG Query Flow

```mermaid
sequenceDiagram
    participant User
    participant UI as Streamlit
    participant Ret as Retriever
    participant SF as Snowflake
    participant LLM as Cortex AI

    User->>UI: Ask question
    UI->>Ret: similarity_search(question, k=5)
    activate Ret
    Ret->>SF: AI_EMBED('model', question)
    SF-->>Ret: query vector
    Ret->>SF: VECTOR_COSINE_SIMILARITY(query_vector, stored_vectors) TOP 5
    SF-->>Ret: top-k chunks + metadata
    Ret-->>UI: Document objects
    deactivate Ret

    UI->>LLM: CORTEX.COMPLETE(model, context + question)
    activate LLM
    LLM-->>UI: Generated answer
    deactivate LLM

    UI-->>User: Answer + source citations
```

### Document Ingestion Flow

```mermaid
sequenceDiagram
    participant Script as load_documents.py
    participant Part as Partitioner
    participant SF as Snowflake

    Script->>Script: Scan source_docs/ for files
    loop For each document
        Script->>Part: partition_and_chunk(file)
        activate Part
        Part->>Part: Detect format (PDF/DOCX/MD/HTML)
        Part->>Part: partition_* + chunk_by_title
        Part->>Part: Extract metadata
        Part-->>Script: List[Chunk]
        deactivate Part

        Script->>SF: INSERT INTO staging (chunks without vectors)
        Script->>SF: INSERT INTO final SELECT *, AI_EMBED(chunk_text) FROM staging
        Script->>SF: DELETE FROM staging WHERE source_name = file
    end

    Script->>Script: Log summary (success/failed counts)
```

### API Request Sequence

```mermaid
sequenceDiagram
    participant Client
    participant API as REST API
    participant Auth as Auth Service
    participant DB as Database

    Client->>API: POST /shipments {data}
    API->>Auth: Validate token
    Auth-->>API: 200 OK (valid)
    API->>DB: INSERT INTO shipments
    DB-->>API: shipment_id
    API-->>Client: 201 Created {shipment_id}

    Note over Client,DB: Error path
    Client->>API: POST /shipments {bad data}
    API->>API: Validate request body
    API-->>Client: 400 Bad Request {errors}
```

## Message Types

| Arrow | Meaning |
|-------|---------|
| `→` (solid) | Synchronous message (caller waits) |
| `-->` (dashed) | Return / response |
| `->>`  (solid, open head) | Asynchronous message (fire and forget) |
| `-->>` (dashed, open head) | Async response |

## Advanced Elements

### Alt / Opt / Loop Fragments

```mermaid
sequenceDiagram
    participant P as Pipeline
    participant SF as Snowflake

    P->>SF: Check if document exists
    alt Document exists (incremental mode)
        SF-->>P: Found — skip
    else New document
        P->>SF: Load chunks to staging
        P->>SF: Generate embeddings
        P->>SF: Move to final table
    end

    loop For each failed file
        P->>P: Log error, continue
    end
```

### Parallel Execution

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant Lint as sqlfluff
    participant Test as dbt test

    par Lint and Test in parallel
        CI->>Lint: sqlfluff lint models/
        Lint-->>CI: lint results
    and
        CI->>Test: dbt test --target ci
        Test-->>CI: test results
    end
```

## When to Use Sequence Diagrams

| Situation | Why |
|-----------|-----|
| **API design** | Document request/response flow before implementation |
| **Pipeline debugging** | Trace the exact order of operations |
| **Integration documentation** | Show how services interact |
| **Error flow documentation** | Map what happens when things fail |
| **Onboarding** | Explain multi-step processes visually |

## Tools

| Tool | Format | Integration |
|------|--------|-------------|
| **Mermaid** | Markdown code blocks | Obsidian, GitHub, GitLab, Notion |
| **PlantUML** | Text-based DSL | IntelliJ, VS Code, CI pipelines |
| **draw.io** | Visual drag-and-drop | Confluence, standalone |
| **Lucidchart** | Visual SaaS | Team collaboration |
| **swimlanes.io** | Text-based, web | Quick browser-based diagrams |

Mermaid is recommended for this vault — renders natively in Obsidian.

## Notation Deep Dive

### Actors and Participants

In Mermaid sequence diagrams, `participant` creates a box and `actor` creates a stick figure. Use `actor` for human roles and `participant` for system components:

```mermaid
sequenceDiagram
    actor Analyst as Data Analyst
    participant API as REST API
    participant DW as Snowflake
    Analyst->>API: Request dataset
    API->>DW: SELECT * FROM gold.report
    DW-->>API: Result set
    API-->>Analyst: JSON response
```

### Synchronous vs Asynchronous Messages

| Syntax | Rendering | Semantics |
|--------|-----------|-----------|
| `->>` | Solid line, filled arrowhead | Synchronous — caller blocks until response |
| `-->>` | Dashed line, filled arrowhead | Return / response message |
| `-)` | Solid line, open arrowhead | Asynchronous — fire and forget |
| `--)` | Dashed line, open arrowhead | Async response / callback |

Asynchronous messages are critical for modelling event-driven and streaming architectures where the producer does not wait for the consumer.

### Activation Bars

Activation bars (the narrow rectangles on lifelines) show when a participant is actively processing. Use `activate` / `deactivate` or the shorthand `+` / `-` syntax:

```
Caller->>+Service: request
Service-->>-Caller: response
```

### Combined Fragments

| Fragment | Purpose | Data Engineering Use Case |
|----------|---------|--------------------------|
| `alt` / `else` | Conditional branching | Incremental vs full load decision |
| `opt` | Optional execution | Skip processing if no new data |
| `loop` | Repeated execution | Paginated API calls, batch iteration |
| `par` / `and` | Parallel execution | Concurrent pipeline stages |
| `critical` | Atomic section | Transaction boundaries |
| `break` | Exit early | Circuit breaker tripped |

## REST API Pagination Flow

A common data engineering pattern — extracting all pages from a paginated API:

```mermaid
sequenceDiagram
    participant Ingest as Ingestion Script
    participant API as REST API
    participant S3 as S3 Bucket

    Ingest->>API: GET /records?page=1&limit=500
    activate API
    API-->>Ingest: 200 OK {data, next_page: 2, total_pages: 47}
    deactivate API
    Ingest->>S3: PUT page_1.json

    loop While next_page exists
        Ingest->>API: GET /records?page=N&limit=500
        activate API
        API-->>Ingest: 200 OK {data, next_page}
        deactivate API
        Ingest->>S3: PUT page_N.json

        opt Rate limit hit (429)
            Ingest->>Ingest: Backoff (exponential delay)
        end
    end

    Ingest->>Ingest: Log extraction complete (pages, row count)
```

Key details this diagram captures:
- The **loop fragment** makes the pagination strategy explicit
- The **opt fragment** documents rate-limit handling without cluttering the main flow
- Raw files land in **S3** before any transformation — separation of concerns

## dbt Run Orchestration Flow

How an orchestrator coordinates a full dbt pipeline run with logging:

```mermaid
sequenceDiagram
    participant Sched as Airflow Scheduler
    participant Worker as Airflow Worker
    participant dbt as dbt Core
    participant SF as Snowflake
    participant Log as T0 Pipeline Log

    Sched->>Worker: Trigger DAG run
    activate Worker

    Worker->>dbt: dbt source freshness
    activate dbt
    dbt->>SF: SELECT MAX(_loaded_at) FROM sources
    SF-->>dbt: freshness results
    deactivate dbt

    alt Sources are fresh
        Worker->>dbt: dbt run --select staging+
        activate dbt
        dbt->>SF: CREATE OR REPLACE TABLE stg_*
        SF-->>dbt: success
        dbt->>SF: MERGE INTO dim_* / fact_*
        SF-->>dbt: rows affected
        deactivate dbt

        Worker->>dbt: dbt test --select staging+
        activate dbt
        dbt->>SF: Run assertions (not_null, unique, relationships)
        SF-->>dbt: test results
        dbt-->>Worker: pass / fail summary
        deactivate dbt

        Worker->>Log: INSERT run metadata (status, duration, row counts)
    else Sources are stale
        Worker->>Log: INSERT stale source alert
        Worker-)Sched: Mark DAG as skipped
    end

    deactivate Worker
```

## Kafka Producer-Consumer Flow

Modelling asynchronous message passing through a broker:

```mermaid
sequenceDiagram
    participant App as Application Service
    participant Prod as Kafka Producer
    participant Broker as Kafka Broker
    participant CG as Consumer Group
    participant DB as Analytics Database

    App-)Prod: Emit domain event (async)
    activate Prod
    Prod->>Broker: Produce to topic (key, value, headers)
    Broker-->>Prod: ACK (offset assigned)
    deactivate Prod

    Note over Broker: Message persisted to partition log

    Broker-)CG: Poll returns batch of messages
    activate CG
    CG->>CG: Deserialise and validate schema
    alt Valid message
        CG->>DB: INSERT INTO events table
        DB-->>CG: Committed
        CG->>Broker: Commit offset
    else Schema validation failure
        CG->>Broker: Produce to dead-letter topic
        CG->>Broker: Commit offset (skip poison pill)
    end
    deactivate CG
```

Notable modelling choices:
- The initial `App-)Prod` uses an **async arrow** — the application does not wait for Kafka acknowledgement
- The **dead-letter topic** pattern is shown via the `alt` fragment — critical for production pipelines
- **Offset commits** are explicit — this documents the at-least-once delivery guarantee

## CI/CD Deployment Flow

A typical data engineering CI/CD pipeline from git push to production:

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant Git as GitLab
    participant CI as CI Runner
    participant Lint as sqlfluff
    participant dbt as dbt
    participant SF as Snowflake CI Schema
    participant Prod as Snowflake Production
    participant Slack as Slack

    Dev->>Git: git push (feature branch)
    Git-)CI: Webhook trigger
    activate CI

    par Static Checks
        CI->>Lint: sqlfluff lint models/
        Lint-->>CI: Lint results
    and
        CI->>dbt: dbt compile --target ci
        dbt-->>CI: Compilation check
    end

    alt Lint or compile fails
        CI-->>Git: Pipeline failed
        CI-)Slack: Notify: build failed
    else All checks pass
        CI->>dbt: dbt run --target ci
        activate dbt
        dbt->>SF: Build models in CI schema
        SF-->>dbt: Success
        deactivate dbt

        CI->>dbt: dbt test --target ci
        activate dbt
        dbt->>SF: Run test suite
        SF-->>dbt: All tests pass
        deactivate dbt

        CI-->>Git: Pipeline passed (ready for merge)
    end

    deactivate CI

    Note over Dev,Git: After merge to main

    Git-)CI: Merge trigger (main branch)
    activate CI
    CI->>dbt: dbt run --target prod
    activate dbt
    dbt->>Prod: Deploy models to production
    Prod-->>dbt: Success
    deactivate dbt
    CI-)Slack: Notify: deployment complete
    deactivate CI
```

## When to Use Sequence Diagrams in Data Engineering

Sequence diagrams excel in specific documentation scenarios. Choosing the right diagram type saves time and improves clarity.

### Strong Use Cases

| Scenario | Why Sequence Diagrams Work |
|----------|---------------------------|
| **API integration documentation** | Shows the exact request/response handshake, authentication flow, pagination, and error handling in order |
| **Troubleshooting production incidents** | Trace the timeline of what called what, when, and what failed — invaluable during post-mortems |
| **Onboarding new team members** | Multi-step orchestration flows (Airflow DAGs, CI/CD pipelines) are far clearer as sequences than as prose |
| **Contract negotiation with external teams** | Agree on interaction protocol before writing code — the diagram becomes the specification |
| **Async architecture documentation** | Kafka/pub-sub flows need the temporal ordering that [[Data Flow Diagrams|DFDs]] cannot express |

### When Not to Use Them

- **Data lineage** — use a [[Data Flow Diagrams|DFD]] instead; sequence diagrams do not show data at rest
- **Infrastructure topology** — use an architecture diagram; sequence diagrams do not show deployment concerns
- **Simple single-step operations** — a sequence diagram for "client calls API, gets response" adds no value over prose

### Tips for Effective Sequence Diagrams

1. **Limit participants to 5-7** — more than that and the diagram becomes unreadable; split into multiple diagrams
2. **Show the error path** — use `alt` fragments to document what happens when things fail, not just the happy path
3. **Label messages precisely** — "send data" is useless; "POST /api/v2/shipments {manifest_id, items[]}" is useful
4. **Use activation bars consistently** — they show processing duration and help identify bottlenecks
5. **Add notes for context** — `Note over` blocks explain *why* something happens without cluttering the message flow

---

## Error Handling and Retry Flows

### Pipeline Retry with Exponential Backoff

```mermaid
sequenceDiagram
    participant Orch as Orchestrator
    participant Job as Pipeline Job
    participant API as External API
    participant Log as Audit Log

    Orch->>Job: Execute ingestion
    activate Job

    Job->>API: GET /data?page=1
    API-->>Job: 503 Service Unavailable

    loop Retry with exponential backoff (attempt 1..N)
        Note over Job: Wait 2^attempt seconds (2s, 4s, 8s...)
        Job->>API: GET /data?page=1 (retry)
        alt Success
            API-->>Job: 200 OK {data}
            Job->>Log: INSERT retry_success (attempts, total_delay)
        else Max retries exceeded
            API-->>Job: 503 Service Unavailable
            Job->>Log: INSERT retry_exhausted (last_error, attempts)
            Job->>Orch: Raise ExtractionError
        end
    end

    deactivate Job
```

### Dead-Letter Queue Flow

```mermaid
sequenceDiagram
    participant Prod as Producer
    participant Broker as Kafka Broker
    participant Consumer as Consumer
    participant DLQ as Dead-Letter Topic
    participant Alert as Alerting Service
    participant Ops as Ops Dashboard

    Prod->>Broker: Produce message (main topic)
    Broker-)Consumer: Poll batch
    activate Consumer

    alt Valid message
        Consumer->>Consumer: Deserialise and validate
        Consumer->>Consumer: Process and transform
        Consumer->>Broker: Commit offset
    else Deserialisation failure
        Consumer->>DLQ: Produce to dead-letter topic (original payload + error metadata)
        Consumer->>Broker: Commit offset (skip poison pill)
    else Processing failure (after retries)
        Consumer->>DLQ: Produce to dead-letter topic (payload + stack trace)
        Consumer->>Broker: Commit offset
    end

    deactivate Consumer

    Note over DLQ: Dead-letter messages accumulate

    DLQ-)Alert: Threshold breach (> N messages in window)
    Alert->>Ops: Page on-call engineer
    Ops->>DLQ: Inspect failed messages
    Ops->>Broker: Replay corrected messages to main topic
```

### Circuit Breaker State Transitions

```mermaid
sequenceDiagram
    participant Client as Pipeline Client
    participant CB as Circuit Breaker
    participant Svc as Downstream Service
    participant Mon as Monitoring

    Note over CB: State: CLOSED (normal operation)

    Client->>CB: Request
    CB->>Svc: Forward request
    Svc-->>CB: 500 Internal Server Error
    CB->>CB: Increment failure counter

    Client->>CB: Request
    CB->>Svc: Forward request
    Svc-->>CB: 500 Internal Server Error
    CB->>CB: Failure threshold reached

    Note over CB: State: OPEN (fail fast)
    CB->>Mon: Circuit opened (service: downstream, failures: 5)

    Client->>CB: Request
    CB-->>Client: CircuitOpenError (no call to service)

    Note over CB: After cooldown period (e.g. 30s)
    Note over CB: State: HALF-OPEN (probe)

    Client->>CB: Request
    CB->>Svc: Forward probe request
    alt Probe succeeds
        Svc-->>CB: 200 OK
        Note over CB: State: CLOSED (reset counters)
        CB->>Mon: Circuit closed (service recovered)
        CB-->>Client: Success
    else Probe fails
        Svc-->>CB: 500 Error
        Note over CB: State: OPEN (restart cooldown)
        CB->>Mon: Circuit remains open
        CB-->>Client: CircuitOpenError
    end
```

### Webhook Delivery with Signature Verification

```mermaid
sequenceDiagram
    participant Src as Source System
    participant Wh as Webhook Endpoint
    participant Auth as Signature Validator
    participant Queue as Processing Queue
    participant DLQ as Dead-Letter Queue

    Src->>Src: Generate HMAC-SHA256(payload, shared_secret)
    Src->>Wh: POST /webhooks/events (payload + X-Signature header)
    activate Wh

    Wh->>Auth: Validate signature
    activate Auth
    Auth->>Auth: Recompute HMAC-SHA256(request_body, stored_secret)
    Auth->>Auth: Compare with X-Signature (constant-time)

    alt Signature valid
        Auth-->>Wh: Verified
        Wh->>Queue: Enqueue event for processing
        Wh-->>Src: 202 Accepted
    else Signature invalid
        Auth-->>Wh: Rejected
        Wh->>DLQ: Log rejected payload (source IP, timestamp)
        Wh-->>Src: 401 Unauthorised
    else Signature missing
        Auth-->>Wh: Missing header
        Wh-->>Src: 400 Bad Request
    end

    deactivate Auth
    deactivate Wh
```
