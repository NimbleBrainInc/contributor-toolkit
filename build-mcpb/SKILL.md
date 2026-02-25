---
name: build-mcpb
description: Build MCP servers end-to-end. Scaffolds a production-ready Python or TypeScript server from API documentation, implements tools, validates the MCPB bundle, and authors companion skills. Covers the full lifecycle from API analysis to PR-ready deliverable. Use when building a new MCP server, wrapping an API, or creating an integration. Triggers include "build an MCP server", "create a server for X", "/build-mcpb".
license: Apache-2.0
compatibility: Python 3.13+, uv, ruff, ty OR Node.js 24+, npm. Docker, mpak CLI. Claude Code with filesystem access.
allowed-tools: Read Write Bash Glob Grep WebFetch AskUserQuestion
metadata:
  tags:
    - mcp
    - server
    - mcpb
    - api
    - scaffolding
    - skills
    - typescript
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
    - typescript
    - node
    - zod
    - vitest
    - skill-generation
  version: "0.1.0"
  surfaces:
    - claude-code
  author:
    name: NimbleBrain
    url: https://nimblebrain.ai
  examples:
    - prompt: Build an MCP server for the Notion API
      context: Starting a new Python integration from API docs
    - prompt: Create a TypeScript MCP server for the GitHub API
      context: Building a new TypeScript server from API docs
---

# Build MCPB

Build MCP servers end-to-end: scaffold from API docs, implement tools, validate the bundle, and author companion skills. Supports Python (FastMCP) and TypeScript (@modelcontextprotocol/sdk).

## Quick Start

```
> /build-mcpb
```

The skill auto-detects language and service name from the current directory.

## Pipeline Overview

```
Phase 0: Detect           Auto-detect language + service name from repo
Phase 1: API Analysis     Fetch docs, identify resources, propose tools
Phase 2: Scaffold         Verify project structure from template
Phase 3: Implement        Write tool logic, models, client
Phase 4: Verify           Lint, typecheck, test
Phase 5: Validate Bundle  Manifest, build, bundle, MTF scan, runtime
Phase 6: Author Skills    Generate 2-3 companion skills
Phase 7: Prepare PR       Assemble PR with server + skills
```

## Phase 0: Detect

Auto-detect the project language and service name. The contributor has already created the repo from the correct template via `nimblebrain-contributor`.

### 0a: Language

Check the current working directory:
- `pyproject.toml` exists → **Python**
- `package.json` exists → **TypeScript**
- Neither → ask the user which language they're using
- Both → ask the user (unusual — clarify which is primary)

### 0b: Service Name

1. If `manifest.json` exists, parse the `name` field and strip the scope: `@nimblebraininc/<name>` → `<name>`
2. Otherwise, derive from the directory name: strip `mcp-` prefix (e.g., `mcp-stripe` → `stripe`)
3. If neither works, ask the user

### 0c: Naming Variables

Derive all naming from the service name:

| Variable | Python | TypeScript |
|----------|--------|------------|
| Package scope | `@nimblebraininc/<name>` | `@nimblebraininc/<name>` |
| Module / package | `mcp_<name>` (underscores) | `mcp-<name>` (hyphens) |
| Source directory | `src/mcp_<name>/` | `src/` |
| Env var | `<NAME>_API_KEY` | `<NAME>_API_KEY` |

### 0d: Confirm

Show detection results and ask the user to confirm before proceeding:

```
=> Detected:
   Language: [Python/TypeScript]
   Service: <name>
   Module: <module>
   Env var: <NAME>_API_KEY

=> Correct? [Y/n]
```

## Phase 1: API Analysis

1. **Fetch and analyze** the API documentation (use WebFetch)
2. **Identify**:
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

## Phase 2: Scaffold

The template repo already created the project structure. Verify it's intact.

**Python** — check that these exist:
- `src/mcp_<name>/server.py`, `api_client.py`, `api_models.py`, `__init__.py`
- `tests/`
- `manifest.json`, `pyproject.toml`, `Makefile`
- `.github/workflows/ci.yml`, `.github/workflows/build-bundle.yml`

**TypeScript** — check that these exist:
- `src/index.ts`, `config.ts`, `constants.ts`, `types.ts`, `schemas.ts`, `formatters.ts`
- `src/handlers/`, `src/utils/apiClient.ts`, `src/utils/errorResponse.ts`
- `tests/`
- `manifest.json`, `server.json`, `package.json`, `tsconfig.json`, `Makefile`
- `.github/workflows/ci.yml`, `.github/workflows/build-bundle.yml`

If any files are missing, create them following the patterns in `references/PATTERNS.md`.

See `references/PATTERNS.md` for full directory structure reference.

## Phase 3: Implement

### Concept Mapping

| Concept | Python | TypeScript |
|---------|--------|------------|
| Response models | Pydantic `BaseModel` | Zod `z.object()` in `types.ts` |
| Tool input validation | Python type hints + FastMCP | Zod schemas in `schemas.ts` |
| HTTP client | `aiohttp.ClientSession` | `fetch()` built-in |
| Tool registration | `@mcp.tool()` decorator | `server.registerTool()` call |
| Response formatting | Return Pydantic model directly | `formatters.ts` → JSON.stringify |
| Error handling | `try/except APIError` | `try/catch` → `errorResponse()` |

### If Python

Implement in this order:
1. **`api_models.py`** — Pydantic models for API responses. Use `Field(alias=...)` for camelCase mapping.
2. **`api_client.py`** — Async aiohttp client. Set BASE_URL, add one method per endpoint.
3. **`server.py`** — FastMCP server with `@mcp.tool()` decorators. Global client with lazy init. Dual transport (http_app + stdio).

See `references/PATTERNS.md` → "Python Server Patterns" for complete code examples.

### If TypeScript

Implement in this order:
1. **`src/types.ts`** — Zod schemas for API response shapes
2. **`src/utils/apiClient.ts`** — Rename class, set BASE_URL, add methods. Update import in `errorResponse.ts`.
3. **`src/schemas.ts`** — Zod input schema for each tool
4. **`src/formatters.ts`** — One formatter per resource type (pure functions)
5. **`src/handlers/<tool>.ts`** — One file per tool
6. **`src/index.ts`** — `server.registerTool()` for each tool
7. **`src/config.ts`** — Rename env var
8. **`manifest.json`** + **`server.json`** — Fill all TODO fields, then `make sync`

See `references/PATTERNS.md` → "TypeScript Server Patterns" for complete code examples.

### Critical Notes (TypeScript)

- Use **exact dependency versions** (no `^` or `~`) — range specifiers are L2 MTF findings
- Use `.js` extensions in all imports (Node ESM requirement)
- Never edit `src/constants.ts` manually — use `make sync`
- Never edit `.github/workflows/` — shared infrastructure
- Skills must be **self-contained** in a single `SKILL.md` (no `references/` subdirectory)

## Phase 4: Verify

Run all checks and fix any issues before proceeding.

**Python:**
```bash
uv sync --dev
uv run ruff format src/ tests/
uv run ruff check src/ tests/
uv run ty check src/
uv run pytest tests/ -v
```

**TypeScript:**
```bash
make check
```
This runs: `format:check` → `lint` → `typecheck` → `test`

## Phase 5: Validate Bundle

### 5a: Manifest Validation

Check `manifest.json` against MCPB v0.4 spec. See `references/CONVENTIONS.md` for the full manifest format per language.

Key checks:
- Scoped naming: `@nimblebraininc/<name>`
- `user_config` entries have `sensitive: true` for API keys
- **Python:** `server.type: "python"`, `entry_point` is module path
- **TypeScript:** `server.type: "node"`, `entry_point: "build/index.js"`

### 5b: Build Validation

- **Python:** `uv sync` succeeds, entry point module is importable
- **TypeScript:** `npm run build` succeeds, `build/index.js` exists

### 5c: Bundle Inspection

- **Python:** `mcpb build` produces clean bundle
- **TypeScript:** `make bundle` (builds, prunes dev deps, packs)
- Both: no accidental large files (.git, node_modules), manifest.json present in bundle root

### 5d: MTF Compliance (if mpak-scanner available)

```bash
mpak-scanner scan .
```

### 5e: Runtime Validation

**Python:**
```bash
echo '{"jsonrpc":"2.0","method":"tools/list","id":1}' | \
  uv run python -m mcp_<name>.server
```

**TypeScript:**
```bash
echo '{"jsonrpc":"2.0","method":"tools/list","id":1}' | \
  node build/index.js --stdio
```

Both: server responds with valid JSON-RPC `tools/list` result. No garbage on stdout (logs go to stderr).

## Phase 6: Author Companion Skills

1. **Analyze the server's tools**:
   - **Python:** Extract all `@mcp.tool()` functions from `server.py`
   - **TypeScript:** Extract all `server.registerTool()` calls from `src/index.ts`

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

**Python servers:**
```
skills/<skill-name>/
├── SKILL.md
└── references/
    └── SERVER_TOOLS.md
```

**TypeScript servers:**
```
skills/<skill-name>/
└── SKILL.md              # Self-contained — no subdirectories
```

5. **Validate each skill:**
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

## References

See `references/` in this skill for:
- `CONVENTIONS.md` — Naming, manifest format, versioning, build system, entry points (Python + TypeScript)
- `PATTERNS.md` — Complete code patterns, directory structures, CI workflows (Python + TypeScript)
- `SKILL_FORMAT.md` — Skill frontmatter specification and validation rules

Python templates are also available in `templates/` within this skill folder.

For TypeScript, the canonical patterns are in the `NimbleBrainInc/mcp-server-template-typescript` GitHub template repo — already cloned when the project was created.
