# Markdown to Notebook Conversion

Pattern for converting documentation markdown (with SQL code blocks) into Jupyter/Snowflake notebooks.

## Use Case

You have a `QUERIES.md` file with documented SQL examples:

````markdown
## Semantic Search

Find chunks most similar to a question:

```sql
SELECT source_name, chunk_text,
    VECTOR_COSINE_SIMILARITY(AI_EMBED('model', 'my question'), vector) AS score
FROM knowledge_base_embeddings
ORDER BY score DESC LIMIT 5;
```

## Metadata Query

List all ingested documents:

```sql
SELECT DISTINCT source_name, source_type, COUNT(*) AS chunks
FROM knowledge_base_embeddings
GROUP BY 1, 2;
```
````

You want to convert this into a Snowflake Notebook (`.ipynb`) where each SQL block becomes an executable cell and each prose section becomes a markdown cell.

## Conversion Script

```python
#!/usr/bin/env python3
"""Convert a markdown file with SQL blocks into a Jupyter notebook."""

import re
import json
import nbformat
from pathlib import Path

def markdown_to_notebook(md_path: str, output_path: str, setup_sql: str = None):
    text = Path(md_path).read_text(encoding='utf-8')

    nb = nbformat.v4.new_notebook()
    nb.metadata['kernelspec'] = {
        'display_name': 'SQL',
        'language': 'sql',
        'name': 'sql',
    }

    # Optional setup cell (USE DATABASE, USE SCHEMA)
    if setup_sql:
        nb.cells.append(nbformat.v4.new_code_cell(source=setup_sql))

    # Split on ```sql ... ``` blocks
    parts = re.split(r'```sql\s*\n(.*?)\n```', text, flags=re.DOTALL)

    for i, part in enumerate(parts):
        content = part.strip()
        if not content:
            continue

        if i % 2 == 0:
            # Even indices = markdown (prose between SQL blocks)
            nb.cells.append(nbformat.v4.new_markdown_cell(source=content))
        else:
            # Odd indices = SQL code
            nb.cells.append(nbformat.v4.new_code_cell(source=content))

    nbformat.write(nb, Path(output_path))
    print(f"Notebook written: {output_path} ({len(nb.cells)} cells)")


if __name__ == '__main__':
    markdown_to_notebook(
        md_path='docs/05-QUERIES.md',
        output_path='docs/06-WORKBOOK.ipynb',
        setup_sql='USE DATABASE KNOWLEDGE_DB;\nUSE SCHEMA KNOWLEDGE;',
    )
```

## How It Works

1. Read the markdown file
2. Split on `` ```sql ... ``` `` blocks using regex
3. Even-indexed parts → markdown cells (prose)
4. Odd-indexed parts → code cells (SQL)
5. Optionally prepend a setup cell (`USE DATABASE`, `USE SCHEMA`)
6. Write as nbformat v4 notebook

## Importing into Snowflake

The generated `.ipynb` can be imported directly into **Snowflake Notebooks** (Snowsight > Notebooks > Import):

- SQL cells execute against your Snowflake warehouse
- Markdown cells render as documentation
- No local Jupyter install needed

## Make Target

```makefile
workbook:
	python scripts/queries_to_workbook.py
```

## Variations

### Python Code Blocks

Extend the regex to handle both SQL and Python:

```python
# Split on ```sql or ```python blocks
parts = re.split(r'```(sql|python)\s*\n(.*?)\n```', text, flags=re.DOTALL)
```

### Table of Contents Cell

Add a TOC as the first markdown cell by extracting `##` headings:

```python
headings = re.findall(r'^##\s+(.+)$', text, re.MULTILINE)
toc = "## Contents\n\n" + "\n".join(f"- {h}" for h in headings)
nb.cells.insert(0, nbformat.v4.new_markdown_cell(source=toc))
```
