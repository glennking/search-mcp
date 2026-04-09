# search-mcp Design Spec

**Date:** 2026-04-09
**Status:** Approved
**Related Issue:** glennking/research#18

## Overview

A self-hosted MCP server providing AI-optimized web search and content extraction. Powered by SearXNG as the search backend. No third-party API keys required. Modeled after Tavily's design but fully self-hostable.

## Goals

- Replace unreliable DuckDuckGo MCP with a robust, self-hosted alternative
- Provide structured search results optimized for LLM consumption
- Extract clean markdown content from web pages
- Single `podman compose up` to run everything

## Non-Goals (MVP)

- Crawling (multi-page traversal)
- Sitemap generation
- Deep research (multi-query synthesis)
- Result caching
- JavaScript-rendered page support (future enhancement)

## MCP Tools Interface

### Transport

HTTP/SSE transport. Server exposes an SSE endpoint at `http://localhost:3691/sse`. Claude Code connects via URL in MCP config — no stdio, no `docker exec`.

### `search(query, num_results=10, include_content=False)`

**Parameters:**
- `query` (string, required) — search terms
- `num_results` (int, default 10, max 30) — number of results to return
- `include_content` (bool, default False) — fetch and include extracted page content for each result

**Returns:** JSON array:
```json
[
  {
    "title": "Page Title",
    "url": "https://example.com/page",
    "snippet": "Brief excerpt from search engine...",
    "score": 0.95,
    "content": "Full markdown content..." // null if include_content=False
  }
]
```

When `include_content=True`, the server queries SearXNG first, then fires off concurrent extraction calls for each result URL via `asyncio.gather()`. Individual extraction failures return `null` content for that result — they do not fail the entire search.

### `extract(url)`

**Parameters:**
- `url` (string, required) — page to extract

**Returns:** JSON object:
```json
{
  "url": "https://example.com/page",
  "title": "Page Title",
  "content": "Clean markdown content...",
  "content_length": 4523
}
```

## Architecture

### Approach

Async single-process Python server using the MCP Python SDK with SSE transport. Async I/O via `asyncio` + `httpx` enables concurrent page extraction when `include_content=True`.

### Project Structure

```
src/
  search_mcp/
    __init__.py
    server.py          # MCP server setup, tool registration, SSE endpoint
    tools/
      __init__.py
      search.py        # search() tool — queries SearXNG, optional content extraction
      extract.py       # extract() tool — fetches URL, returns clean markdown
    clients/
      __init__.py
      searxng.py       # async HTTP client for SearXNG JSON API
      extractor.py     # trafilatura wrapper for content extraction
    config.py          # settings from env vars
```

- `server.py` — entrypoint. Creates MCP server, registers tools, starts SSE listener.
- Tools are thin — validate input, call clients, format output.
- Clients handle actual HTTP and extraction logic.
- `config.py` reads environment variables with sensible defaults.

### Data Flow

```
1. Claude Code sends MCP tool call over SSE
2. MCP server receives request
3a. search(): query SearXNG JSON API → format results → optionally extract content concurrently → return
3b. extract(): fetch URL via httpx → pass to trafilatura → return clean markdown
4. Claude Code receives structured JSON response
```

## SearXNG Configuration

SearXNG runs as a Docker container with a custom `settings.yml` mounted from `config/searxng/`.

- **Default engines:** Google, Bing, DuckDuckGo
- **Output format:** JSON (internal API only)
- **Rate limiting:** Disabled (local use only)
- **UI:** Disabled (API-only usage)
- **Safe search:** Off (research tool, not consumer-facing)

The MCP server queries `http://searxng:8080/search?q=...&format=json` on the internal Docker network.

To change engines, edit `config/searxng/settings.yml` directly — it's checked into the repo and mounted into the container.

## Docker Compose

```yaml
services:
  searxng:
    image: searxng/searxng:latest
    volumes:
      - ./config/searxng:/etc/searxng:rw
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080"]
      interval: 5s
      timeout: 3s
      retries: 5
    # ports:
    #   - "8888:8080"  # uncomment for debugging

  search-mcp:
    build: .
    ports:
      - "${MCP_PORT:-3691}:3691"
    depends_on:
      searxng:
        condition: service_healthy
    environment:
      - SEARXNG_URL=http://searxng:8080
      - MCP_PORT=3691
```

- SearXNG is internal only — no host port by default
- MCP server exposes port 3691 for Claude Code
- Health check ensures SearXNG is ready before MCP server starts querying
- Claude Code MCP config: `{"url": "http://localhost:3691/sse"}`
- Single `podman compose up` starts everything

## Configuration

All via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_PORT` | `3691` | Port for SSE endpoint |
| `SEARXNG_URL` | `http://searxng:8080` | SearXNG base URL |
| `SEARCH_TIMEOUT` | `10` | SearXNG query timeout (seconds) |
| `EXTRACT_TIMEOUT` | `15` | Per-URL extraction timeout (seconds) |
| `DEFAULT_NUM_RESULTS` | `10` | Default results per search |
| `MAX_NUM_RESULTS` | `30` | Maximum allowed results |

No config files beyond SearXNG's `settings.yml`. Environment variables are the only knob.

## Error Handling

- **SearXNG unreachable:** `"Search backend unavailable. Is SearXNG running?"` with hint to check `podman compose ps`
- **No results:** Return empty array `[]`, not an error
- **Extraction fetch fails** (timeout, 403, DNS): `"Failed to extract content from <url>: 403 Forbidden"`
- **Extraction returns empty** (JS-heavy, paywall): Return response with `"content": ""` and `"note": "Extraction returned no content. Page may require JavaScript."`
- **Invalid parameters** (missing query, num_results > 30): Validation error before hitting any backend
- **`search()` with `include_content=True`:** Individual extraction timeouts return null content for that result; the overall search still succeeds

**Timeouts:**
- SearXNG query: 10 seconds
- Content extraction per URL: 15 seconds

## Testing Strategy

### Unit Tests (no containers required)
- `search()` correctly passes params to SearXNG client (mocked)
- `extract()` returns clean markdown from sample HTML (mocked)
- Config loads env vars with correct defaults
- Error handling returns structured errors for each failure mode
- Parameter validation rejects invalid inputs

### Integration Tests (require `podman compose up`)
- `search()` returns real results from SearXNG
- `search()` with `include_content=True` returns content for results
- `extract()` on a known stable URL returns expected content

### MCP Protocol Tests
- Tool discovery (list tools returns search + extract)
- Tool call/response round-trip over SSE

Run unit tests: `python3 -m pytest tests/unit/`
Run all tests: `podman compose up -d && python3 -m pytest tests/`

## Dependencies

### Python
- `mcp` — MCP Python SDK (SSE transport)
- `httpx` — async HTTP client (SearXNG queries, page fetching)
- `trafilatura` — web content extraction
- `pytest` / `pytest-asyncio` — testing

### Infrastructure
- Podman + Podman Compose
- SearXNG Docker image (`searxng/searxng:latest`)

## Future Enhancements

- `crawl(url)` — multi-page site traversal with depth control
- `map(url)` — sitemap generation
- `research(query)` — multi-query deep research with synthesized report
- Result caching (in-memory or SQLite)
- Playwright fallback for JS-rendered pages
