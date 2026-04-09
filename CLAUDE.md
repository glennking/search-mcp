# CLAUDE.md

## Project Overview

search-mcp — Self-hosted MCP server for AI-optimized web search and content extraction. Powered by SearXNG.

| Directory | Purpose | Tech |
|-----------|---------|------|
| `src/` | MCP server source code | Python |
| `tests/` | Test files | pytest |
| `docs/` | Documentation | Markdown |
| `scripts/` | Build and utility scripts | Shell |

## Session Start

When a conversation begins without a specific task — or when I ask "what's next", "what should I work on", or similar — do the following:

1. Check the GitHub project board (https://github.com/users/glennking/projects/19) for open issues, grouped by status and priority
2. Summarize what's in progress, what's approved and ready, and what's in scoping
3. Note any issues assigned to me or recently updated
4. Ask what I want to pick up or continue

This gives me a quick standup-style briefing without needing to context-switch to the browser.

## Common Commands

```bash
# Run the MCP server locally
podman compose up

# Run tests
python3 -m pytest tests/

# Pre-commit hooks (trufflehog secret scanning)
pre-commit install                    # One-time setup
pre-commit run trufflehog --all-files # Manual scan
```

## Architecture

```
Claude Code ──MCP (stdio)──▶ search-mcp-server (Python, Docker)
                                  │
                                  ├── search(query) ──▶ SearXNG (Docker) ──▶ Google/Bing/DDG
                                  │
                                  └── extract(url) ──▶ trafilatura
```

Both services run in Docker Compose (using Podman).

## Key Patterns

- MCP server uses the Python `mcp` SDK with stdio transport
- SearXNG provides the search backend — no third-party API keys needed
- trafilatura handles web content extraction to clean markdown
- All services containerized via Podman Compose

## Coding Rules

- **No hardcoded paths.** Use env vars, config, or relative paths.
- **UTC timestamps everywhere.** Store and compare times in UTC.
- **Use Podman, not Docker.** Use `podman` and `podman compose` instead of `docker` and `docker compose`.
- **Use `python3` explicitly.** Never use bare `python` or `pip`.
- **No third-party API keys required.** The entire stack must be self-hostable without external service dependencies.

## Dev Workflow

All work is issue-driven. Follow this lifecycle for every change:

### 1. Find next issue
- Query the GitHub project board (https://github.com/users/glennking/projects/19) for the highest-priority unstarted issue
- Present it to the user for confirmation before starting

### 2. Scope the issue
- Move issue to **Scoping** on the project board
- Determine if this issue changes product scope: does it add, remove, or modify what the product does (vs. how it's built)?
  - If yes: add the `prd-update` label
- Assign PRD requirement IDs — either map to existing IDs in `docs/prd.md` or create new ones
  - New requirements get the next available ID in their section (e.g., SEC-017 follows SEC-016)
  - Add a `## PRD Requirements` section to the issue body listing the IDs
- If `prd-update` label: update `docs/prd.md` with the new/changed requirements and bump the version
- Size the issue (XS/S/M/L/XL) if not already sized
- Move issue to **Approved** when scoping is complete
- Not every issue needs deep scoping — bugs, maintenance, and small fixes can move through quickly

### 3. Start work
- Use `start_work_session` MCP tool — it handles all of this in one call:
  - Updates the issue status to **In Progress** on the project board
  - Creates a worktree + branch (e.g., `issue-{N}-short-name`)
  - Logs activity start with the issue context
  - Adds `in-progress` label to the issue
  - Returns the `activity_id` — **save this for step 9**
- Worktrees live as sibling directories to the main repo
- All implementation happens in the worktree, not the main clone

### 4. Implement
- Work entirely within the worktree
- **Commit between changes** — don't batch everything. Commit after implementation, after review fixes, after test fixes
- Reference the issue number in commits

### 5. Self-review
- Review the full diff (`git diff main..HEAD`) before opening a PR
- Check for: bugs, missing edge cases, circular imports, fields not threaded through all layers, lint errors
- Fix issues in separate commits (don't amend — keep the review trail)

### 6. Update docs
- Update `docs/architecture.md` with any architectural changes **before pushing** — this prevents forgetting
- Add new ADRs for significant design decisions (Context, Decision, Rationale)

### 7. Test
- Run tests for regressions
- Write targeted tests for new code when the change adds testable logic
- Run linter on changed files
- Test edge cases and fallback behavior

### 8. Push and open PR
- Push branch and create PR with:
  - **Summary** of changes (bullet points)
  - **Review findings** — bugs caught during self-review and how they were fixed
  - **Test results** — what was tested and the outcome
- The PR body should tell the full story, not just list files changed

### 9. Merge and clean up
- Squash merge to main
- **End the work session** using `end_work_session` with the `activity_id` from step 3 — this completes the activity, removes labels, and updates status
- Remove the worktree and delete the local branch
- Pull main: `git pull`
- **Close the issue explicitly**: `gh issue close {N} -c "summary of what was done"`
- Update the project board status to **Done**

**For quick work without a worktree** (docs updates, process changes): use `log_activity_start` / `log_activity_complete` directly since there's no worktree or labels to manage.

### PRD maintenance
- `docs/prd.md` is the product specification — "what should this product be"
- Issues with the `prd-update` label change product scope and require a PRD update during Scoping (step 2)
- Issues without `prd-update` (bugs, maintenance, quality) don't touch the PRD but should still reference existing requirement IDs where applicable
- New requirement IDs follow the next sequential number in their PRD section
- When closing a `prd-update` issue, bump the PRD version (e.g., 1.0 → 1.1)

### Unplanned fixes (errors discovered mid-work)

When you hit a bug or blocker unrelated to the current issue:

1. **Do not fix it inline** — it muddies the current issue's PR
2. **Create a new issue** describing the problem
3. **Stash or set aside** the current work (worktree makes this easy — just switch directories)
4. **Work the new issue** through the normal lifecycle (worktree → implement → review → PR → merge)
5. **Resume the original issue** once the fix is merged

If the fix is truly trivial (one-line typo, config value) and blocking the current work, it's acceptable to commit directly to main — but still create a retroactive issue to document what happened and why.

### Maintenance and cleanup

When you notice tech debt (lint warnings, deprecated patterns, dead code) during feature work:

1. **Don't fix it in the feature branch** — it inflates the diff and mixes concerns
2. **Create a `maintenance` issue** after the feature merges, with specifics
3. **Work it through the normal lifecycle** when prioritized

Maintenance issues default to P2. Use the `maintenance` label.

### Key principles
- **Review before PR, not after** — catch bugs before they're visible
- **PR tells the story** — review findings and test results are as valuable as the diff
- **Docs before close** — architecture doc must reflect the current state before an issue is closed
- **One issue at a time** — finish the lifecycle before starting the next issue
- **Every change has an issue** — even unplanned fixes and maintenance get tracked
- **Don't mix concerns** — feature branches do features, maintenance branches do cleanup
- **Worktrees over branches** — each issue gets its own directory; no stashing, no context loss, parallel work is trivial

### GitHub Project
- Project URL: https://github.com/users/glennking/projects/19
- Status field values: Suggested, **Scoping**, Approved, In progress, In review, Done, Declined
- Priority field: P0 (critical) through P3 (nice-to-have)
- Always update status when starting (In progress) and finishing (Done) an issue
