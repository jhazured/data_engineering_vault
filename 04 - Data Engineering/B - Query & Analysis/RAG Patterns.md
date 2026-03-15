# RAG (Retrieval-Augmented Generation) Patterns

RAG combines information retrieval with LLM generation — the model answers questions using context retrieved from your own data rather than relying solely on its training data.

## Core Pipeline

```
User Question
      │
      ▼
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Embed Query │────▶│  Vector Search    │────▶│  Assemble Prompt │
│  AI_EMBED()  │     │  COSINE_SIMILARITY│     │  Context + Query  │
└─────────────┘     └──────────────────┘     └────────┬────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │  LLM Generation  │
                                              │  CORTEX.COMPLETE │
                                              └─────────────────┘
                                                       │
                                                       ▼
                                                   Answer + Sources
```

## 1. Retrieval — Semantic Search

Embed the user's question at query time and find the most similar stored chunks:

```sql
SELECT
    source_name,
    section,
    chunk_text,
    metadata,
    VECTOR_COSINE_SIMILARITY(
        AI_EMBED('snowflake-arctic-embed-m-v1.5', %(query)s),
        vector
    ) AS score
FROM knowledge_base_embeddings
ORDER BY score DESC
LIMIT %(k)s
```

**Key decisions:**
- **k value** (number of chunks): 3-5 for focused answers, 8-10 for comprehensive coverage
- **Similarity threshold**: Optionally filter `WHERE score > 0.7` to exclude weak matches
- **Embedding model**: Must be the same model used at ingestion time

## 2. Context Assembly

Concatenate retrieved chunks into a single context string for the prompt:

```python
def _build_context(docs, max_docs=4):
    """Truncate to top-k docs to stay within model context window."""
    context_parts = []
    for doc in docs[:max_docs]:
        source = doc.metadata.get('source_name', 'unknown')
        section = doc.metadata.get('section', '')
        context_parts.append(
            f"[Source: {source} | Section: {section}]\n{doc.page_content}"
        )
    return "\n\n---\n\n".join(context_parts)
```

**Why truncate?** LLM context windows have limits. Including too many chunks dilutes relevance and increases cost/latency.

## 3. Prompt Construction

Structure the prompt with clear instructions, context, and the question:

```python
RAG_PROMPT = """You are a knowledgeable assistant. Answer the question using
ONLY the context provided below. If the context doesn't contain enough
information, say so — do not make up an answer.

CONTEXT:
{context}

QUESTION: {question}

ANSWER:"""
```

**Best practices:**
- Instruct the model to use only provided context (reduces hallucination)
- Include source attribution instructions if needed
- Keep the system prompt concise — most of the token budget should go to context

## 4. Generation

Call the LLM with the assembled prompt:

```python
def personal_mistral(question, docs, model='mistral-large2'):
    context = _build_context(docs)
    prompt = RAG_PROMPT.format(context=context, question=question)

    sql = "SELECT SNOWFLAKE.CORTEX.COMPLETE(%s, %s) AS response"
    result = run_sql(sql, params=(model, prompt))
    return result[0]['RESPONSE']
```

## LangChain-Compatible Retriever

Implement the retriever interface so it works with LangChain pipelines:

```python
from langchain_core.documents import Document

class SnowflakeRetriever:
    def __init__(self, table='knowledge_base_embeddings', embed_model='snowflake-arctic-embed-m-v1.5'):
        self.table = table
        self.embed_model = embed_model

    def similarity_search(self, query: str, k: int = 5) -> list[Document]:
        k = max(1, min(k, 20))  # Clamp to safe range
        rows = self._run_vector_search(query, k)
        return [
            Document(
                page_content=row['CHUNK_TEXT'],
                metadata={
                    'source_name': row['SOURCE_NAME'],
                    'section': row['SECTION'],
                    'score': row['SCORE'],
                    **row.get('METADATA', {}),
                }
            )
            for row in rows
        ]
```

## Direct Q&A vs RAG

| Mode | When to Use | How |
|------|------------|-----|
| **Direct Q&A** | General knowledge questions | `CORTEX.COMPLETE(model, question)` — no retrieval |
| **RAG** | Questions about your data/docs | Retrieve → build context → `CORTEX.COMPLETE(model, prompt)` |

## RAG Quality Tuning

| Lever | Effect | Trade-off |
|-------|--------|-----------|
| **k (chunks retrieved)** | More context = more comprehensive | More noise, slower, higher cost |
| **Chunk size** | Larger = more context per chunk | Less precise retrieval |
| **Chunk overlap** | Prevents splitting key info | More storage, redundancy |
| **Similarity threshold** | Filters weak matches | May miss relevant but dissimilar chunks |
| **Model choice** | Larger = better reasoning | Slower, more expensive |
| **max_docs in prompt** | Controls context window usage | Too many = diluted, too few = incomplete |

## Chunking Strategy by Document Type

| Document Type | Recommended Chunk Size | Why |
|--------------|----------------------|-----|
| Books / narrative | 2000 chars | Long-form context needs larger windows |
| Business policies | 800 chars | Precise, self-contained clauses |
| API docs / runbooks | 1000-1500 chars | Procedural steps with moderate context |
| Code documentation | 1500 chars | Function/class scope boundaries |

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Better Approach |
|-------------|-------------|-----------------|
| No source attribution | Users can't verify answers | Include source metadata in response |
| Stuffing entire documents | Exceeds context window, dilutes relevance | Chunk and retrieve top-k |
| Same embedding model for all formats | Different content types embed differently | Use consistent model but tune chunk sizes |
| No similarity threshold | Low-relevance chunks pollute context | Filter or warn on low scores |
| Hardcoding model names in SQL | SQL injection risk | Validate against allowlist |
