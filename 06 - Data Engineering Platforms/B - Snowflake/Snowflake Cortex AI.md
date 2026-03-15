# Snowflake Cortex AI

Snowflake Cortex provides LLM inference and embedding generation natively inside Snowflake — no external API calls, data stays within your security perimeter.

## Core Functions

### CORTEX.COMPLETE() — Text Generation

```sql
SELECT SNOWFLAKE.CORTEX.COMPLETE('mistral-large2', 'Explain data lakehouse architecture')
    AS response;
```

With RAG context:

```sql
SELECT SNOWFLAKE.CORTEX.COMPLETE(
    'mistral-large2',
    'Using the following context, answer the question.\n\nCONTEXT:\n' ||
    chunk_text || '\n\nQUESTION: What is the data flow?\n\nANSWER:'
) AS response
FROM knowledge_base_embeddings
WHERE source_name = 'architecture-guide.pdf'
ORDER BY VECTOR_COSINE_SIMILARITY(AI_EMBED('snowflake-arctic-embed-m-v1.5', 'data flow'), vector) DESC
LIMIT 1;
```

### AI_EMBED() — Vector Embeddings

```sql
-- Embed text at query time
SELECT AI_EMBED('snowflake-arctic-embed-m-v1.5', 'How do I optimize query performance?')
    AS query_vector;

-- Embed during ingestion
INSERT INTO knowledge_base_embeddings (chunk_text, vector)
SELECT chunk_text, AI_EMBED('snowflake-arctic-embed-m-v1.5', chunk_text)
FROM knowledge_base_staging;
```

### VECTOR_COSINE_SIMILARITY() — Semantic Search

```sql
SELECT chunk_text,
    VECTOR_COSINE_SIMILARITY(
        AI_EMBED('snowflake-arctic-embed-m-v1.5', 'my question'),
        vector
    ) AS score
FROM knowledge_base_embeddings
ORDER BY score DESC
LIMIT 5;
```

## Available Models

| Model | Type | Best For |
|-------|------|----------|
| `mistral-large2` | Generation | High-quality general answers |
| `mixtral-8x7b` | Generation | Fast, good quality |
| `llama3.1-70b` | Generation | Open-source alternative |
| `llama3.1-8b` | Generation | Fast, lightweight |
| `snowflake-arctic` | Generation | Snowflake-optimised |
| `snowflake-arctic-embed-m-v1.5` | Embedding | 768-dim text embeddings |
| `e5-base-v2` | Embedding | Alternative embeddings |

## RBAC Setup

### Create Cortex Roles

```sql
USE ROLE SECURITYADMIN;

-- Role for text generation
CREATE DATABASE ROLE IF NOT EXISTS CORTEX_USER;
GRANT DATABASE ROLE CORTEX_USER TO ROLE ENGINEER;
GRANT DATABASE ROLE CORTEX_USER TO ROLE ANALYST;

-- Role for embedding generation (ingestion pipelines)
CREATE DATABASE ROLE IF NOT EXISTS CORTEX_EMBED_USER;
GRANT DATABASE ROLE CORTEX_EMBED_USER TO ROLE ENGINEER;
```

### Grant Cortex Functions

```sql
USE ROLE ACCOUNTADMIN;

-- Grant COMPLETE (generation) to CORTEX_USER
GRANT USAGE ON FUNCTION SNOWFLAKE.CORTEX.COMPLETE(VARCHAR, VARCHAR)
    TO DATABASE ROLE CORTEX_USER;

-- Grant AI_EMBED to CORTEX_EMBED_USER
GRANT USAGE ON FUNCTION SNOWFLAKE.CORTEX.AI_EMBED(VARCHAR, VARCHAR)
    TO DATABASE ROLE CORTEX_EMBED_USER;
```

## SQL Injection Prevention

Model names are passed as SQL parameters but appear in function calls — validate against an allowlist:

```python
CORTEX_MODELS = frozenset({
    'mistral-large2', 'mixtral-8x7b', 'snowflake-arctic',
    'llama3.1-70b', 'llama3.1-8b', 'llama3.1-405b',
})

def safe_model_name(model: str) -> str:
    """Validate model name to prevent SQL injection."""
    model = model.strip().lower()
    if model not in CORTEX_MODELS:
        raise ValueError(f"Unknown model: {model}. Allowed: {sorted(CORTEX_MODELS)}")
    return model
```

**Why?** `CORTEX.COMPLETE(model, prompt)` takes the model as a string parameter. If user-supplied model names flow into SQL without validation, it's an injection vector.

Similarly, validate database/schema/warehouse names:

```python
import re

def safe_id(name: str) -> str:
    """Validate Snowflake identifier (alphanumeric + underscore only)."""
    if not re.match(r'^[A-Za-z_][A-Za-z0-9_]*$', name):
        raise ValueError(f"Invalid identifier: {name}")
    return name
```

## Python Integration

```python
def cortex_complete(question, model='mistral-large2'):
    model = safe_model_name(model)
    sql = "SELECT SNOWFLAKE.CORTEX.COMPLETE(%s, %s) AS response"
    result = run_sql(sql, params=(model, question))
    return result[0]['RESPONSE']

def cortex_rag(question, docs, model='mistral-large2'):
    context = "\n\n".join(doc.page_content for doc in docs[:4])
    prompt = f"Answer using ONLY the context below.\n\nCONTEXT:\n{context}\n\nQUESTION: {question}"
    return cortex_complete(prompt, model)
```

## Cost Considerations

- Cortex functions consume **Snowflake credits** (compute, not per-token)
- Embedding is cheaper than generation
- Larger models (llama3.1-405b) use significantly more compute
- Use smaller models (llama3.1-8b, mixtral-8x7b) for development/testing
- `AI_EMBED` at ingestion time (batch) is more efficient than per-query embedding of stored text
