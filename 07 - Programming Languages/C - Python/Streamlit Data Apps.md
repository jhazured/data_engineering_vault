# Streamlit Data Apps

Patterns for building interactive data applications with Streamlit — commonly used for internal tools, dashboards, and RAG interfaces.

## Basic Structure

```python
import streamlit as st

st.set_page_config(page_title="Knowledge Base", layout="wide")
st.title("Knowledge Base Search")

# Sidebar controls
with st.sidebar:
    model = st.selectbox("Model", ["mistral-large2", "mixtral-8x7b", "llama3.1-8b"])
    k = st.slider("Chunks to retrieve", min_value=1, max_value=10, value=5)

# Main form
with st.form("search_form"):
    question = st.text_area("Ask a question:", height=100)
    submitted = st.form_submit_button("Search")
```

## Session State Management

Streamlit reruns the entire script on every interaction. Use `st.session_state` to persist data across reruns:

```python
# Initialise state
if 'answer' not in st.session_state:
    st.session_state.answer = None
    st.session_state.docs = []
    st.session_state.error = None

# Set state on form submission
if submitted and question:
    try:
        docs = retriever.similarity_search(question, k=k)
        answer = cortex_rag(question, docs, model=model)
        st.session_state.answer = answer
        st.session_state.docs = docs
        st.session_state.error = None
    except Exception as e:
        st.session_state.error = str(e)

# Display persisted results (survives Streamlit reruns)
if st.session_state.error:
    st.error(st.session_state.error)
elif st.session_state.answer:
    st.markdown(st.session_state.answer)
```

## Displaying Retrieved Sources

Show source metadata alongside the answer:

```python
if st.session_state.docs:
    st.subheader("Sources")
    for doc in st.session_state.docs:
        meta = doc.metadata
        score = meta.get('score', 0)
        with st.expander(f"{meta.get('source_name', 'Unknown')} — {meta.get('section', '')} (score: {score:.3f})"):
            st.markdown(doc.page_content)
            if meta.get('page_number'):
                st.caption(f"Page {meta['page_number']}")
```

## Safe Model Selection

Use a dropdown with a fixed allowlist — never let users type model names that flow into SQL:

```python
ALLOWED_MODELS = ["mistral-large2", "mixtral-8x7b", "llama3.1-70b", "llama3.1-8b"]
model = st.selectbox("Cortex Model", ALLOWED_MODELS)
```

## Common Patterns

### Progress Indicators

```python
with st.spinner("Searching knowledge base..."):
    docs = retriever.similarity_search(question, k=k)

with st.spinner("Generating answer..."):
    answer = cortex_rag(question, docs, model=model)
```

### Columns Layout

```python
col1, col2 = st.columns([2, 1])
with col1:
    st.markdown(answer)
with col2:
    st.metric("Chunks Retrieved", len(docs))
    st.metric("Top Score", f"{docs[0].metadata['score']:.3f}" if docs else "N/A")
```

### Caching

Cache expensive operations (database connections, reference data):

```python
@st.cache_resource
def get_retriever():
    return SnowflakeRetriever()

@st.cache_data(ttl=3600)
def get_document_list():
    return run_sql("SELECT DISTINCT source_name FROM knowledge_base_embeddings")
```

## Deployment

```bash
# Local development
streamlit run app_streamlit.py

# With custom port
streamlit run app_streamlit.py --server.port 8501

# Snowflake Streamlit (Streamlit in Snowflake)
# Deploy via Snowsight > Streamlit > + Streamlit App
```

## Testing Patterns

Streamlit apps are hard to unit test directly. Extract business logic into separate modules:

```
app_streamlit.py          # UI only — forms, display, session state
scripts/core/retriever.py  # Testable — similarity_search()
scripts/core/agent.py      # Testable — cortex_rag()
```

Test the core modules with pytest; test the UI manually or with Streamlit's `AppTest` (experimental).
