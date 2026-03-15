# Python Testing with pytest

Testing patterns for Python data engineering projects — unit tests, integration tests with mocked databases, and real-file validation.

## Test Organisation

```
tests/
├── conftest.py              # Shared fixtures, path setup
├── test_chunking.py         # Unit tests (no external deps)
├── test_pipeline.py         # Integration tests (mock Snowflake)
└── test_sample_docs.py      # Real-file tests (graceful skip)
```

## conftest.py — Path Setup

Ensure imports work from the test directory:

```python
# tests/conftest.py
import sys
from pathlib import Path

# Add project root and scripts/ to sys.path
project_root = Path(__file__).parent.parent
sys.path.insert(0, str(project_root))
sys.path.insert(0, str(project_root / 'scripts'))
```

## Unit Tests (No External Dependencies)

Test pure logic without databases, APIs, or file I/O:

```python
# test_chunking.py
import os
import pytest
from ingest.chunker import _chunk_config, _looks_like_heading, _get_section_title
from ingest.models import Chunk

class TestChunkConfig:
    def test_defaults(self):
        config = _chunk_config()
        assert config['max_characters'] == 2000
        assert config['overlap'] == 300

    def test_env_override(self, monkeypatch):
        monkeypatch.setenv('CHUNK_MAX_CHARS', '800')
        config = _chunk_config()
        assert config['max_characters'] == 800

    def test_clamping(self, monkeypatch):
        monkeypatch.setenv('CHUNK_MAX_CHARS', '50')  # Below minimum
        config = _chunk_config()
        assert config['max_characters'] >= 100  # Clamped

class TestHeadingDetection:
    @pytest.mark.parametrize("text,expected", [
        ("Chapter 1: Introduction", True),
        ("Part III — Advanced Topics", True),
        ("ARCHITECTURE OVERVIEW", True),       # Short + uppercase
        ("This is a regular paragraph that goes on for a while...", False),
    ])
    def test_heading_patterns(self, text, expected):
        assert _looks_like_heading(text) == expected

class TestChunkModel:
    def test_fields(self):
        c = Chunk(chunk_text="hello", section="Intro", source_type="pdf",
                  source_name="doc.pdf", metadata={'page': 1})
        assert c.source_type == "pdf"
        assert c.metadata['page'] == 1
```

**Key patterns:**
- `monkeypatch.setenv()` — override environment variables per test (auto-reverted)
- `@pytest.mark.parametrize` — run same test with multiple inputs
- No mocking needed — testing pure functions

## Integration Tests (Mock External Services)

Test pipeline logic with mocked Snowflake connections:

```python
# test_pipeline.py
from unittest.mock import patch, MagicMock

class TestIncrementalLoad:
    @patch('scripts.core.snowflake_helper.snowflake_run_new')
    def test_incremental_skips_existing(self, mock_run):
        # Simulate: document already exists in final table
        mock_run.return_value = [{'SOURCE_NAME': 'existing.pdf'}]

        result = load_documents(mode='incremental', source_dir='sample_docs/')

        # Verify it checked for existing docs
        calls = [str(c) for c in mock_run.call_args_list]
        assert any('SELECT DISTINCT source_name' in c for c in calls)

    @patch('scripts.core.snowflake_helper.snowflake_run_new')
    def test_staging_cleanup_on_error(self, mock_run):
        # Simulate: embedding INSERT fails
        mock_run.side_effect = [
            None,                    # staging INSERT succeeds
            Exception("embed fail"), # embedding INSERT fails
            None,                    # staging DELETE (cleanup)
        ]

        with pytest.raises(Exception, match="embed fail"):
            load_single_file('test.pdf')

        # Verify staging was cleaned up despite error
        delete_calls = [c for c in mock_run.call_args_list if 'DELETE' in str(c)]
        assert len(delete_calls) >= 1

class TestSnowflakeRetriever:
    @patch('scripts.core.snowflake_helper.snowflake_run_new')
    def test_returns_documents(self, mock_run):
        mock_run.return_value = [{
            'CHUNK_TEXT': 'Some content',
            'SOURCE_NAME': 'doc.pdf',
            'SECTION': 'Chapter 1',
            'SCORE': 0.85,
            'METADATA': {'page_number': 3},
        }]

        retriever = SnowflakeRetriever()
        docs = retriever.similarity_search("test query", k=5)

        assert len(docs) == 1
        assert docs[0].page_content == 'Some content'
        assert docs[0].metadata['score'] == 0.85

    def test_k_clamping(self):
        retriever = SnowflakeRetriever()
        assert retriever._clamp_k(0) == 1
        assert retriever._clamp_k(100) == 20
        assert retriever._clamp_k(5) == 5
```

## Real-File Tests (Graceful Skipping)

Test actual file parsing but skip if libraries or sample files are missing:

```python
# test_sample_docs.py
import pytest
from pathlib import Path

SAMPLE_DIR = Path(__file__).parent.parent / 'sample_docs'

def has_samples():
    return SAMPLE_DIR.exists() and any(SAMPLE_DIR.iterdir())

@pytest.mark.skipif(not has_samples(), reason="sample_docs/ not present")
class TestSampleDocuments:
    def test_partition_markdown(self):
        try:
            from ingest.markdown_partitioner import partition_and_chunk_markdown
        except ImportError:
            pytest.skip("unstructured not installed")

        md_files = list(SAMPLE_DIR.glob('*.md'))
        assert len(md_files) > 0

        chunks = partition_and_chunk_markdown(md_files[0])
        assert len(chunks) > 0
        assert all(c.source_type == 'markdown' for c in chunks)
        assert all(c.chunk_text.strip() for c in chunks)  # No empty chunks

    def test_partition_pdf(self):
        try:
            from ingest.pdf_partitioner import partition_and_chunk_pdf
        except ImportError:
            pytest.skip("unstructured[pdf] not installed")

        pdf_files = list(SAMPLE_DIR.glob('*.pdf'))
        if not pdf_files:
            pytest.skip("No PDF samples")

        chunks = partition_and_chunk_pdf(pdf_files[0])
        assert len(chunks) > 0
        assert all('chunk_index' in c.metadata for c in chunks)
```

**Key pattern:** `pytest.skip()` for graceful degradation — tests pass in lean CI environments where heavy dependencies aren't installed.

## Fixtures

Reusable test setup:

```python
# conftest.py
@pytest.fixture
def sample_chunks():
    return [
        Chunk(chunk_text="First chunk", section="Intro",
              source_type="pdf", source_name="test.pdf", metadata={}),
        Chunk(chunk_text="Second chunk", section="Body",
              source_type="pdf", source_name="test.pdf", metadata={}),
    ]

@pytest.fixture
def mock_snowflake():
    with patch('scripts.core.snowflake_helper.snowflake_run_new') as mock:
        yield mock
```

## Running Tests

```bash
# All tests
pytest tests/ -v --tb=short

# Specific test file
pytest tests/test_chunking.py -v

# Specific test class/method
pytest tests/test_pipeline.py::TestIncrementalLoad::test_staging_cleanup -v

# With coverage
pytest tests/ --cov=scripts --cov-report=term-missing

# Only unit tests (skip slow integration tests)
pytest tests/ -m "not integration" -v

# Make target
make test
```

## Marking Tests

```python
@pytest.mark.integration
def test_snowflake_connection():
    """Requires live Snowflake — skip in CI without credentials."""
    ...

@pytest.mark.slow
def test_large_pdf_parsing():
    """Takes >30s — skip in quick feedback loops."""
    ...
```

```ini
# pytest.ini or pyproject.toml
[tool:pytest]
markers =
    integration: requires Snowflake connection
    slow: takes >30 seconds
```
