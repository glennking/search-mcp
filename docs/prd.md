# search-mcp — Product Requirements Document

**Version:** 1.0

## Overview

A self-hosted MCP server that provides AI-optimized web search and content extraction. Designed as a Tavily alternative with no third-party API key dependencies, powered by SearXNG.

## Requirements

### Core Features
- CORE-001: MCP server exposes tools via stdio transport for use with Claude Code and other MCP clients
- CORE-002: `search(query)` tool returns structured, LLM-optimized results (title, URL, snippet, content)
- CORE-003: `extract(url)` tool returns clean markdown content from any web page
- CORE-004: Search results are ranked and deduplicated across search engines
- CORE-005: Content extraction strips navigation, ads, and boilerplate — returns only meaningful content

### Infrastructure
- INFRA-001: SearXNG provides the search backend, running in Docker
- INFRA-002: MCP server runs in Docker alongside SearXNG via Docker Compose
- INFRA-003: No third-party API keys required — fully self-hostable
- INFRA-004: Configuration via environment variables or config file

### Quality
- QUAL-001: Search results return within 5 seconds for typical queries
- QUAL-002: Content extraction handles JavaScript-rendered pages gracefully (fallback, not hard failure)
- QUAL-003: Errors return structured error messages, not stack traces

### Future Features
- FUT-001: `crawl(url)` — multi-page site traversal with depth control
- FUT-002: `map(url)` — sitemap generation
- FUT-003: `research(query)` — multi-query deep research with synthesized report
- FUT-004: Result caching to reduce duplicate searches
