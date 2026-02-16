---
name: build-mcpb
description: Build MCP servers end-to-end. Scaffolds a production-ready Python/FastMCP server from API documentation, implements tools, validates the MCPB bundle, and authors companion skills. Covers the full lifecycle from API analysis to PR-ready deliverable. Use when building a new MCP server, wrapping an API, or creating an integration. Triggers include "build an MCP server", "create a server for X", "/build-mcpb".
license: Apache-2.0
compatibility: Requires Python 3.13+, uv, ruff, ty, Docker, mpak CLI. Claude Code with filesystem access.
allowed-tools: Read Write Bash Glob Grep WebFetch AskUserQuestion
metadata:
  tags:
    - mcp
    - server
    - mcpb
    - api
    - scaffolding
    - skills
  category: development
  triggers:
    - build an MCP server
    - create an MCP server for
    - wrap this API as MCP
    - build mcp server for
    - scaffold MCP server
    - /build-mcpb
  keywords:
    - mcp-server
    - fastmcp
    - mcpb
    - api-integration
    - python
    - skill-generation
  version: "0.1.0"
  surfaces:
    - claude-code
  author:
    name: NimbleBrain
    url: https://nimblebrain.ai
  examples:
    - prompt: Build me an MCP server for the Notion API
      context: Starting a new integration from API docs
    - prompt: Create an MCP server that wraps the Linear API
      context: Building a new server for project management
---

# Build MCPB

Build MCP servers end-to-end: scaffold from API docs, implement tools, validate the bundle, and author companion skills. One skill, full pipeline.

## Quick Start

```
> Build me an MCP server for the Notion API
```

## Pipeline Overview

```
Phase 1: API Analysis      Fetch docs, identify resources, propose tools
Phase 2: Scaffold           Generate project (code, tests, CI, manifest)
Phase 3: Implement          Write tool logic, models, client
Phase 4: Verify             Lint, typecheck, test
Phase 5: Validate Bundle    Manifest, build, bundle, MTF scan, runtime
Phase 6: Author Skills      Generate 2-3 companion skills
Phase 7: Prepare PR         Assemble PR with server + skills
```

## Phase 1: API Analysis

1. **Fetch and analyze** the API documentation (use WebFetch)
2. **Identify**:
   - Service name (for naming the server)
   - Authentication method (Bearer token, API key, OAuth)
   - Base URL for API calls
   - Available resources and CRUD operations
   - Pagination patterns
   - Response schemas

3. **Check for OpenAPI/Swagger spec** at common paths:
   - `/openapi.json`, `/swagger.json`, `/api/openapi.json`

4. **Present summary** to user for approval:
   ```
   => Analyzing [Service] API...
      Base URL: https://api.example.com/v1
      Auth: Bearer token
      Resources found: [list]

   => Proposed tools ([count]):
      [Resource]: list_X, get_X, create_X, update_X
      ...

   => Proceed? [Y/n]
   ```

## Phase 2: Scaffold Project

Create the project directory using templates from `templates/`:

```
<server-name>/
├── .github/workflows/
│   ├── build-bundle.yml    # MCPB release workflow
│   └── ci.yml              # Lint/test CI
├── src/mcp_<name>/
│   ├── __init__.py         # Version
│   ├── api_models.py       # Pydantic response models
│   ├── api_client.py       # Async HTTP client
│   └── server.py           # FastMCP tools + entrypoints
├── tests/
│   ├── __init__.py
│   └── test_api_models.py
├── .gitignore
├── .mcpbignore
├── .env.example
├── CLAUDE.md
├── Makefile
├── manifest.json           # MCPB manifest (v0.4)
├── pyproject.toml          # uv + ruff + ty
├── pytest.ini
└── README.md
```

## Phase 3: Generate Code

### api_models.py
- Pydantic models for each API response type
- `Field(alias="camelCase")` for JSON field mapping
- `model_config = {"populate_by_name": True}` for models with aliases that need defaults
- List response wrappers with pagination

### api_client.py
- Async client using `aiohttp`
- Bearer token authentication
- Generic `_request` method with error handling
- One method per API endpoint, returning typed Pydantic models

### server.py
- FastMCP server with `@mcp.tool()` decorators
- Global client instance with lazy initialization
- Comprehensive docstrings with Args/Returns
- Health check endpoint
- Dual transport: `app = mcp.http_app()` and `if __name__ == "__main__": mcp.run()`

## Phase 4: Verify

```bash
uv sync --dev
uv run ruff format src/ tests/
uv run ruff check src/ tests/
uv run ty check src/
uv run pytest tests/ -v
```

Fix any issues before proceeding.

## Phase 5: Validate Bundle

Run validation checks:

### 5a: Manifest Validation
- Check manifest.json against MCPB v0.4 spec
- Verify scoped naming: `@nimblebraininc/<name>`
- Validate user_config entries (API keys marked sensitive)

### 5b: Build Validation
- `uv sync` succeeds
- Entry point module exists and is importable

### 5c: Bundle Inspection
- `mcpb build` produces a clean bundle
- No accidental large files (.git, node_modules)
- manifest.json present in bundle root

### 5d: MTF Compliance (if mpak-scanner available)
```bash
mpak-scanner scan .
```

### 5e: Runtime Validation
```bash
echo '{"jsonrpc":"2.0","method":"tools/list","id":1}' | \
  uv run python -m mcp_<name>.server
```
- Server responds to `tools/list` with valid JSON-RPC
- No garbage on stdout (logs go to stderr)

## Phase 6: Author Companion Skills

1. **Analyze the server's tools**: Extract all `@mcp.tool()` functions
2. **Suggest 3-5 skills** based on tool composition patterns:

| Pattern | Description | Example |
|---------|-------------|---------|
| **Daily digest** | Aggregate + summarize | "Summarize today's updates" |
| **Creator** | Structured content from input | "Create meeting notes from transcript" |
| **Organizer** | Categorize + file items | "Sort docs into databases" |
| **Reporter** | Pull data + format report | "Weekly pipeline status report" |
| **Monitor** | Check conditions + alert | "Flag stale deals" |
| **Enricher** | Augment data with context | "Enrich contacts with company info" |

3. **Present suggestions** to user for selection
4. **Generate each skill** as a directory:

```
skills/<skill-name>/
├── SKILL.md              # Frontmatter + instructions
└── references/
    └── SERVER_TOOLS.md   # Quick reference of available tools
```

5. **Validate each skill**:
```bash
mpak skill validate ./skills/<skill-name>
mpak skill pack ./skills/<skill-name>
```

### Skill Quality Checklist
- [ ] Composes 2+ tools (not a single-tool wrapper)
- [ ] Clear trigger (when would someone use this?)
- [ ] Specific tool names from the server
- [ ] Output format defined
- [ ] Example included
- [ ] Declares server dependency in compatibility

## Phase 7: Prepare PR

**PR title:** `Add <server-name> MCP server + companion skills`

**PR body:**
```markdown
## Summary
- New MCP server for <API name> with <N> tools
- <N> companion skills for common workflows

## Server Tools
- `tool_1` - description
- ...

## Skills
- `skill-1` - description
- ...

## Checklist
- [ ] 5+ tools implemented
- [ ] manifest.json valid (v0.4)
- [ ] Tests passing
- [ ] CI passing (lint, format, typecheck, test, bundle, scan)
- [ ] Scanner passes (no critical/high findings)
- [ ] 2+ companion skills with proper frontmatter
- [ ] All skills pass `mpak skill validate`
```

## Key Conventions

### Naming
- Package name: `@nimblebraininc/<name>` (lowercase, hyphens OK)
- Python module: `mcp_<name>` (underscores)
- Environment variable: `<NAME>_API_KEY` (uppercase)

### manifest.json (MCPB v0.4)
```json
{
  "manifest_version": "0.4",
  "name": "@nimblebraininc/<name>",
  "version": "0.1.0",
  "description": "Description here",
  "author": { "name": "NimbleBrain Inc" },
  "user_config": {
    "api_key": {
      "type": "string",
      "title": "API Key",
      "description": "Your API key from...",
      "sensitive": true,
      "required": true
    }
  },
  "server": {
    "type": "python",
    "entry_point": "mcp_<name>.server",
    "mcp_config": {
      "command": "python",
      "args": ["-m", "mcp_<name>.server"],
      "env": {
        "<NAME>_API_KEY": "${user_config.api_key}"
      }
    }
  }
}
```

### Build System
Python servers with src-layout MUST have a `[build-system]` in `pyproject.toml`:
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/mcp_<name>"]
```

### Versioning
All servers stay at v0.x.y. Version lives in four files that MUST stay in sync:

| File | Field |
|------|-------|
| `manifest.json` | `version` |
| `pyproject.toml` | `version` |
| `src/<package>/__init__.py` | `__version__` |

Bump all at once: `make bump VERSION=0.2.0`

## References

See `references/` in this skill for:
- `CONVENTIONS.md` - Package naming, manifest format, user_config, transport patterns
- `PATTERNS.md` - Complete code patterns for server.py, api_client.py, api_models.py, workflows
- `SKILL_FORMAT.md` - Skill frontmatter specification and validation rules

## Templates

Read templates from `templates/` directory in this skill folder. See the directory listing for all available templates.
