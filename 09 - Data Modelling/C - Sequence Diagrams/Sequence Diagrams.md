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
