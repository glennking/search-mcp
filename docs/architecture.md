# search-mcp — Architecture

## Overview

**Languages:** Python
**Frameworks:** MCP Python SDK, trafilatura
**Infrastructure:** Podman Compose (SearXNG + MCP server)

## Components

### MCP Server (`src/`)
- Python application using the `mcp` SDK
- Exposes `search` and `extract` tools via stdio transport
- Communicates with SearXNG over HTTP (internal Docker network)
- Uses trafilatura for content extraction

### SearXNG
- Self-hosted metasearch engine
- Aggregates results from Google, Bing, DuckDuckGo, and others
- Provides JSON API consumed by the MCP server
- Runs as a Docker container

### Docker Compose
- `searxng` service: search backend
- `search-mcp` service: MCP server
- Internal network for service communication
- Stdio transport exposed for Claude Code integration

## Data Flow

```
1. Claude Code sends MCP tool call (search or extract)
2. MCP server receives via stdio
3a. search(): MCP server queries SearXNG JSON API → formats results → returns
3b. extract(): MCP server fetches URL via trafilatura → cleans content → returns markdown
4. Claude Code receives structured response
```

## ADRs

### ADR-001: SearXNG over commercial search APIs
- **Context:** Need a search backend that doesn't require API keys
- **Decision:** Use SearXNG as a self-hosted metasearch engine
- **Rationale:** Free, aggregates multiple engines, well-maintained, JSON API available. Avoids rate limiting and bot detection issues experienced with DDG scraping.

### ADR-002: trafilatura for content extraction
- **Context:** Need to extract clean content from web pages for LLM consumption
- **Decision:** Use trafilatura Python library
- **Rationale:** Purpose-built for web content extraction, handles boilerplate removal, outputs markdown. Lighter than headless browser approach. Can add browser fallback later if needed.

### ADR-003: Podman Compose for deployment
- **Context:** Both SearXNG and the MCP server need containerized deployment
- **Decision:** Use Podman Compose with two services
- **Rationale:** Consistent with user's Podman preference. Keeps services isolated. Single `podman compose up` to start everything.
