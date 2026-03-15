# Model Context Protocol (MCP)

MCP enables AI tools (Claude Code, Claude Desktop) to query external data sources — like Snowflake — live during a conversation.

## What MCP Does

```
Claude Code / Claude Desktop
        │
        ▼  (MCP protocol)
   MCP Server (e.g. snowflake-labs-mcp)
        │
        ▼  (SQL queries)
   Snowflake
        │
        ▼
   Query results returned to Claude
```

Instead of copy-pasting data into prompts, Claude can directly query your Snowflake tables, run semantic searches against your knowledge base, or inspect table schemas — all live.

## Setup

### 1. Create a Snowflake Connection File

`snowflake_connections.toml` (gitignored — contains credentials):

```toml
[default]
account = "xxxxxxx-xxxxxxx"
user = "your_username"
password = "your_password"
warehouse = "TRN_DEV_CENTRAL_WH"
database = "KNOWLEDGE_DB"
schema = "KNOWLEDGE"
role = "ENGINEER"
```

### 2. Configure MCP in Your Project

`.mcp.json` (committed — contains no secrets):

```json
{
  "mcpServers": {
    "snowflake": {
      "command": "uvx",
      "args": [
        "snowflake-labs-mcp",
        "--connections-file",
        "snowflake_connections.toml"
      ]
    }
  }
}
```

### 3. Install the MCP Server

```bash
# uvx auto-installs and runs the package
uvx snowflake-labs-mcp --connections-file snowflake_connections.toml

# Or install manually
pip install snowflake-labs-mcp
```

### 4. Use in Claude Code

Once `.mcp.json` exists in your project root, Claude Code automatically discovers it:

```
> /mcp

# Then in conversation:
"Query the knowledge_base_embeddings table to find all PDF sources"
"Run a semantic search for 'data quality patterns'"
"Show me the schema of the staging tables"
```

### 5. Use in Claude Desktop

Add to Claude Desktop's MCP configuration (Settings > Developer > MCP):

```json
{
  "snowflake": {
    "command": "uvx",
    "args": ["snowflake-labs-mcp", "--connections-file", "/path/to/snowflake_connections.toml"]
  }
}
```

## What Claude Can Do via MCP

| Capability | Example |
|-----------|---------|
| **Query tables** | "How many documents are in the knowledge base?" |
| **Semantic search** | "Find chunks about incremental loading" |
| **Schema inspection** | "What columns does knowledge_base_embeddings have?" |
| **Data analysis** | "Which source documents have the most chunks?" |
| **Pipeline monitoring** | "Show me the last 10 pipeline run log entries" |

## Security Considerations

- `snowflake_connections.toml` contains credentials — always gitignore it
- Use a read-only role for MCP connections when possible
- The MCP server runs locally — queries execute with the configured role's permissions
- No data leaves the Snowflake security perimeter (queries run server-side)

## MCP vs Direct API Calls

| Aspect | MCP | Direct Snowflake API |
|--------|-----|---------------------|
| Authentication | Connection file (local) | Key-pair or OAuth |
| Query interface | Natural language via Claude | SQL via connector |
| Use case | Interactive exploration | Automated pipelines |
| Setup | `.mcp.json` + connection file | Python connector + credentials |
