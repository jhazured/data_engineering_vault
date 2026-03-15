# Document Ingestion & Chunking

Patterns for ingesting multi-format documents (PDF, Word, Markdown, Confluence HTML) into a unified vector-searchable knowledge base.

## Architecture

```
source_docs/
  ├── architecture.pdf
  ├── policy.docx
  ├── runbook.md
  └── confluence-export/
       └── page.html
         │
         ▼
   File Extension Dispatcher
   .pdf  → partition_pdf()
   .docx → partition_docx()
   .md   → partition_markdown()
   .html → partition_confluence()
         │
         ▼
   Unstructured.io partition + chunk_by_title()
         │
         ▼
   Chunk objects with metadata
         │
         ▼
   Staging table (no vectors)
         │
         ▼
   AI_EMBED() → Final table (with vectors)
```

## Unstructured.io — The Abstraction Layer

Unstructured.io provides a single library that handles multiple document formats via `partition_*` functions:

```python
from unstructured.partition.pdf import partition_pdf
from unstructured.partition.docx import partition_docx
from unstructured.partition.md import partition_md
from unstructured.partition.html import partition_html
from unstructured.chunking.title import chunk_by_title
```

### Partition + Chunk Pipeline

```python
# 1. Partition: split document into elements (paragraphs, headings, tables)
elements = partition_pdf(filename=path, strategy='auto')

# 2. Chunk: group elements under headings, respecting size limits
chunks = chunk_by_title(
    elements,
    max_characters=2000,
    overlap=300,
    new_after_n_chars=1500,
    combine_text_under_n_chars=200,
)
```

### Chunking Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `max_characters` | 2000 | Hard limit on chunk size |
| `overlap` | 300 | Characters shared between adjacent chunks |
| `new_after_n_chars` | 1500 | Soft target — start new chunk after this |
| `combine_text_under_n_chars` | 200 | Merge tiny elements into previous chunk |

**Configure via environment:**
```bash
CHUNK_MAX_CHARS=2000      # Books/narrative
CHUNK_MAX_CHARS=800       # Business policies (shorter, self-contained)
CHUNK_OVERLAP=300
```

## Format-Specific Partitioners

### PDF

```python
def partition_and_chunk_pdf(path, strategy='auto'):
    elements = partition_pdf(
        filename=str(path),
        strategy=strategy,                    # 'auto', 'fast', or 'hi_res'
        infer_table_structure=(strategy == 'hi_res'),
    )
    chunks = chunk_by_title(elements, **chunk_config())

    # Extract PDF metadata
    from pypdf import PdfReader
    reader = PdfReader(path)
    info = reader.metadata or {}
    meta = {
        'title': info.get('/Title', path.stem),
        'author': info.get('/Author', 'Unknown'),
        'publication_year': info.get('/CreationDate', '')[:4],
    }

    return [
        Chunk(
            chunk_text=str(c),
            section=get_section_title(c),
            source_type='pdf',
            source_name=path.name,
            metadata={**meta, 'page_number': c.metadata.page_number, 'chunk_index': i},
        )
        for i, c in enumerate(chunks)
    ]
```

**Strategies:**
| Strategy | Speed | Features | Dependencies |
|----------|-------|----------|-------------|
| `fast` | Fast | Basic text extraction | None |
| `auto` | Medium | Layout detection | Default |
| `hi_res` | Slow | Table structure, OCR | tesseract, poppler |

### Word (.docx)

```python
def partition_and_chunk_docx(path):
    elements = partition_docx(filename=str(path))
    chunks = chunk_by_title(elements, **chunk_config())

    from docx import Document as DocxDocument
    doc = DocxDocument(path)
    props = doc.core_properties
    meta = {
        'title': props.title or path.stem,
        'author': props.author or 'Unknown',
        'created': str(props.created) if props.created else None,
        'modified': str(props.modified) if props.modified else None,
    }
    return [Chunk(..., source_type='docx', metadata={**meta, 'chunk_index': i}) for i, c in enumerate(chunks)]
```

### Markdown

```python
def partition_and_chunk_markdown(path):
    elements = partition_md(filename=str(path))
    chunks = chunk_by_title(elements, **chunk_config())

    # Parse YAML front matter
    text = path.read_text(encoding='utf-8').lstrip('\ufeff')  # Handle BOM
    front_matter = parse_front_matter(text)  # Simple regex: key: value lines

    return [Chunk(..., source_type='markdown', metadata={**front_matter, 'chunk_index': i}) ...]
```

### Confluence HTML

```python
def partition_and_chunk_confluence(path):
    elements = partition_html(filename=str(path))
    chunks = chunk_by_title(elements, **chunk_config())

    # Extract Confluence metadata from <meta> tags
    from bs4 import BeautifulSoup
    soup = BeautifulSoup(path.read_text(), 'html.parser')
    meta = {
        'title': (soup.find('meta', {'name': 'page-title'}) or {}).get('content', path.stem),
        'space_name': (soup.find('meta', {'name': 'space-name'}) or {}).get('content', path.parent.name),
        'author': (soup.find('meta', {'name': 'author'}) or {}).get('content'),
    }
    return [Chunk(..., source_type='confluence', metadata={**meta, 'chunk_index': i}) ...]
```

## Heading Detection Heuristics

Identify section boundaries when element metadata is missing:

```python
def looks_like_heading(text):
    """Heuristic heading detection for chunking."""
    text = text.strip()
    if len(text) > 200:
        return False
    if re.match(r'^(Chapter|Part|Section|Appendix)\s', text, re.IGNORECASE):
        return True
    if text.isupper() and len(text) < 100:
        return True
    if text.istitle() and len(text) < 80:
        return True
    return False
```

## Table Handling (hi_res mode)

Tables extracted by Unstructured as HTML are converted to pipe-delimited text for better embedding:

```python
from html.parser import HTMLParser

class TableHTMLParser(HTMLParser):
    """Convert HTML table → 'cell | cell | cell' format."""
    def handle_data(self, data):
        self.current_cell.append(data.strip())
    # ... builds rows of pipe-separated cells

def table_html_to_text(html):
    parser = TableHTMLParser()
    parser.feed(html)
    return "\n".join(" | ".join(row) for row in parser.rows)
```

## Unified Chunk Model

All formats produce the same `Chunk` dataclass:

```python
@dataclass
class Chunk:
    chunk_text: str
    section: str
    source_type: str           # 'pdf', 'docx', 'markdown', 'confluence'
    source_name: str           # Filename
    metadata: dict             # Format-specific (author, page_number, space_name, etc.)
```

This unified schema means a single `knowledge_base_embeddings` table stores all formats, and retrieval works across them seamlessly.

## Incremental vs Full Reload

```python
# Incremental: skip already-loaded documents
existing = run_sql("SELECT DISTINCT source_name FROM knowledge_base_embeddings")
existing_names = {row['SOURCE_NAME'] for row in existing}
files_to_load = [f for f in all_files if f.name not in existing_names]

# Full reload: delete and re-ingest (requires --force flag)
run_sql("DELETE FROM knowledge_base_embeddings WHERE source_name = %s", (name,))
```

## Batch Resilience

Failed files are logged but don't stop the pipeline:

```python
failed = []
for file in files_to_load:
    try:
        chunks = partition_and_chunk(file)
        load_to_snowflake(chunks)
    except Exception as e:
        logger.error(f"Failed: {file.name}: {e}")
        failed.append(file.name)
        continue  # Keep going with remaining files

if failed:
    logger.warning(f"{len(failed)} file(s) failed: {failed}")
```
