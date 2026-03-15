# Vector Embeddings & Semantic Search

## What Are Vector Embeddings?

Embeddings convert text into fixed-length numeric arrays (vectors) that capture semantic meaning. Similar texts produce vectors that are close together in the embedding space, enabling "meaning-based" search rather than keyword matching.

```
"How do I optimize query performance?"  →  [0.12, -0.45, 0.78, ..., 0.33]  (768 dimensions)
"Tips for making SQL queries faster"    →  [0.11, -0.44, 0.79, ..., 0.34]  (very similar vector)
"Company holiday policy"                →  [0.89, 0.12, -0.56, ..., -0.21] (very different vector)
```

## Snowflake Native Vectors

Snowflake supports vectors as a native column type — no external vector database needed.

### Schema

```sql
CREATE TABLE knowledge_base_embeddings (
    source_type   VARCHAR,        -- 'pdf', 'docx', 'markdown', 'confluence'
    source_name   VARCHAR,        -- Filename or document title
    section       VARCHAR,        -- Heading/section within document
    chunk_text    VARCHAR,        -- The actual text chunk
    metadata      VARIANT,        -- JSON: author, page_number, publication_year, etc.
    vector        VECTOR(FLOAT, 768),  -- 768-dimensional embedding
    created_at    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### Generating Embeddings

Use `AI_EMBED()` at insert time to compute vectors:

```sql
INSERT INTO knowledge_base_embeddings (source_type, source_name, section, chunk_text, metadata, vector)
SELECT
    source_type,
    source_name,
    section,
    chunk_text,
    metadata,
    AI_EMBED('snowflake-arctic-embed-m-v1.5', chunk_text)
FROM knowledge_base_staging
```

### Querying — Semantic Search

Embed the query at search time and compare against stored vectors:

```sql
SELECT
    source_name,
    section,
    chunk_text,
    metadata,
    VECTOR_COSINE_SIMILARITY(
        AI_EMBED('snowflake-arctic-embed-m-v1.5', 'How do I optimize query performance?'),
        vector
    ) AS similarity_score
FROM knowledge_base_embeddings
ORDER BY similarity_score DESC
LIMIT 5
```

### Filtered Search

Combine semantic similarity with metadata filters:

```sql
-- Only search within a specific document
WHERE source_name = 'architecture-guide.pdf'

-- Only search PDFs
WHERE source_type = 'pdf'

-- Search with minimum similarity threshold
HAVING similarity_score > 0.7
```

## Embedding Models

| Model | Dimensions | Provider | Use Case |
|-------|-----------|----------|----------|
| `snowflake-arctic-embed-m-v1.5` | 768 | Snowflake (native) | General-purpose, good for docs |
| `e5-base-v2` | 768 | Snowflake Cortex | Alternative general-purpose |
| `text-embedding-3-small` | 1536 | OpenAI | Higher fidelity, external API |

**Critical rule:** The same model must be used for both ingestion and query-time embedding. Mixing models produces meaningless similarity scores.

## Similarity Metrics

| Metric | Range | Best For |
|--------|-------|----------|
| **Cosine similarity** | -1 to 1 (1 = identical) | Normalized text embeddings (most common) |
| **Euclidean distance** | 0 to ∞ (0 = identical) | When magnitude matters |
| **Dot product** | -∞ to ∞ | Pre-normalized vectors |

Snowflake provides `VECTOR_COSINE_SIMILARITY()` — the standard choice for text embeddings.

## Staging → Final Table Pattern

Use a two-table flow for atomic loading with embedding generation:

```
1. Partition document into chunks
2. INSERT chunks into staging table (no vectors)
3. INSERT INTO final SELECT *, AI_EMBED(..., chunk_text) FROM staging
4. TRUNCATE staging (try/finally guarantee)
```

```python
try:
    # Insert raw chunks into staging
    run_sql("INSERT INTO staging (source_type, ...) VALUES (%s, ...)", chunks)

    # Generate embeddings and move to final table
    run_sql("""
        INSERT INTO knowledge_base_embeddings
        SELECT *, AI_EMBED('snowflake-arctic-embed-m-v1.5', chunk_text)
        FROM knowledge_base_staging
        WHERE source_name = %s
    """, (source_name,))
finally:
    # Always clean up staging
    run_sql("DELETE FROM knowledge_base_staging WHERE source_name = %s", (source_name,))
```

**Why two tables?** If embedding generation fails mid-batch, staging can be cleaned up without corrupting the final table.

## Performance Considerations

### Clustering

For large tables, cluster by commonly-filtered columns:

```sql
ALTER TABLE knowledge_base_embeddings
    CLUSTER BY (source_type, source_name);
```

### K-Value Clamping

Always clamp the number of results to prevent abuse:

```python
k = max(1, min(k, 20))  # Safe range: 1-20
```

### Metadata Queries (No Vector Search)

For non-semantic queries (e.g., "list all documents"), skip vector search:

```sql
SELECT DISTINCT source_name, source_type,
    metadata:author::VARCHAR AS author,
    COUNT(*) AS chunk_count
FROM knowledge_base_embeddings
GROUP BY 1, 2, 3
ORDER BY source_name
```

## Snowflake vs External Vector DBs

| Aspect | Snowflake Native | Pinecone/Weaviate/Qdrant |
|--------|-----------------|--------------------------|
| Setup | Zero — native column type | Separate service to provision |
| Security | Within Snowflake perimeter | Separate auth/network config |
| Cost | Included in compute | Separate billing |
| Indexing | No ANN index (brute-force scan) | Purpose-built ANN indexes |
| Scale | Good for <10M vectors | Better for 100M+ vectors |
| Filtering | Full SQL expressiveness | Limited metadata filtering |

**When to use Snowflake native:** Organisational knowledge bases, document search, RAG applications with <10M chunks — keeps everything in one security perimeter.
