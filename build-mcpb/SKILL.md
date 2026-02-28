---
name: build-mcpb
description: Build MCP servers end-to-end. Scaffolds a production-ready Python or TypeScript server from API documentation, implements tools, validates the MCPB bundle, authors companion skills, and guides release to the mpak registry. Covers the full lifecycle from API analysis to published bundle. Use when building a new MCP server, wrapping an API, or creating an integration. Triggers include "build an MCP server", "create a server for X", "/build-mcpb".
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

Build MCP servers end-to-end: scaffold from API docs, implement tools, validate the bundle, author companion skills, and release to the mpak registry. Supports Python (FastMCP) and TypeScript (@modelcontextprotocol/sdk).

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
Phase 7: Release          Commit, push, cut release, verify bundle
```

## Phase 0: Detect

Auto-detect the project language and service name.

### 0a: Prerequisites (cold-start guard)

If the contributor arrived directly (not via `/nimblebrain-contributor`), verify the basics before proceeding:

1. **gh CLI** — `gh auth status` succeeds
2. **Language toolchain** — Python: `uv --version`, `ruff --version`, `ty --version`; TypeScript: `node --version`, `npm --version`
3. **mpak CLI** — `mpak --version` succeeds
4. **Repo structure** — `manifest.json` exists and its `name` starts with `@nimblebraininc/` (confirms it was created from a NimbleBrain template)

If any check fails, tell the contributor what's missing and point them to `/nimblebrain-contributor` or `DEV_SETUP.md` for setup instructions. Don't block on optional tools (e.g., mpak-scanner) — just note they're unavailable and skip the phases that need them.

### 0b: Language

Check the current working directory:
- `pyproject.toml` exists → **Python**
- `package.json` exists → **TypeScript**
- Neither → ask the user which language they're using
- Both → ask the user (unusual — clarify which is primary)

### 0c: Service Name

1. If `manifest.json` exists, parse the `name` field and strip the scope: `@nimblebraininc/<name>` → `<name>`
2. Otherwise, derive from the directory name: strip `mcp-` prefix (e.g., `mcp-stripe` → `stripe`)
3. If neither works, ask the user

### 0d: Naming Variables

Derive all naming from the service name:

| Variable | Python | TypeScript |
|----------|--------|------------|
| Package scope | `@nimblebraininc/<name>` | `@nimblebraininc/<name>` |
| Module / package | `mcp_<name>` (underscores) | `mcp-<name>` (hyphens) |
| Source directory | `src/mcp_<name>/` | `src/` |
| Env var | `<NAME>_API_KEY` | `<NAME>_API_KEY` |

### 0e: Confirm

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

Check `manifest.json` against the mpak registry schema. See `references/CONVENTIONS.md` for the full manifest format per language.

**Registry validation checklist** (read `manifest.json` and verify each):

1. **Name format** — must match `/^@[a-z0-9][a-z0-9-]{0,38}\/[a-z0-9][a-z0-9-]{0,213}$/` (e.g., `@nimblebraininc/stripe`)
2. **Version** — valid semver string (e.g., `0.1.0`)
3. **server.type** — must be one of: `python`, `node`, `binary`
4. **server.mcp_config** — required object with:
   - `command` (string, required) — e.g., `"python"` or `"node"`
   - `args` (array of strings, required) — e.g., `["-m", "mcp_stripe.server"]`
   - `env` (object, optional) — maps env vars to `${user_config.<field>}` references
5. **user_config entries** — each must have:
   - `type` (required) — e.g., `"string"`
   - `sensitive: true` for secrets (API keys, tokens)
   - Referenced via `${user_config.<field>}` in `server.mcp_config.env`
6. **tools array** — each entry needs `name` (required) and `description` (recommended)
7. **Python-specific:** `server.type: "python"`, `entry_point` is module path
8. **TypeScript-specific:** `server.type: "node"`, `entry_point: "build/index.js"`, `${__dirname}` prefix in args

**mpak.json check** — verify `mpak.json` exists in repo root with:
```json
{
  "name": "@nimblebraininc/<name>",
  "maintainers": ["<github-username>"]
}
```
This file is required for package claiming on the registry. The `name` must match `manifest.json`.

### 5b: Build Validation

- **Python:** `uv sync` succeeds, entry point module is importable
- **TypeScript:** `npm run build` succeeds, `build/index.js` exists

### 5c: Bundle Inspection

- **Python:** `make bundle` (vendors deps into `deps/`, packs with `npx @anthropic-ai/mcpb pack`)
- **TypeScript:** `make bundle` (builds, prunes dev deps, packs)
- Both: no accidental large files (.git, node_modules), manifest.json present in bundle root

### 5d: MTF Compliance (if mpak-scanner available)

```bash
mpak-scanner scan .
```

### 5e: Runtime Validation

The MCP protocol requires an initialize handshake before any method calls. Send `initialize`, then `notifications/initialized`, then `tools/list`:

**Python:**
```bash
printf '{"jsonrpc":"2.0","method":"initialize","id":1,"params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}}}\n{"jsonrpc":"2.0","method":"notifications/initialized"}\n{"jsonrpc":"2.0","method":"tools/list","id":2}\n' | uv run python -m mcp_<name>.server 2>/dev/null
```

**TypeScript:**
```bash
printf '{"jsonrpc":"2.0","method":"initialize","id":1,"params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}}}\n{"jsonrpc":"2.0","method":"notifications/initialized"}\n{"jsonrpc":"2.0","method":"tools/list","id":2}\n' | node build/index.js --stdio 2>/dev/null
```

Both: server responds with valid JSON-RPC `initialize` result followed by `tools/list` result. No garbage on stdout (logs go to stderr).

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

## Phase 7: Release

The contributor created and owns the repo — there is no PR to open. The goal is: code on `main` → GitHub Release → `build-bundle.yml` triggers → bundles built and published to the mpak registry.

### 7a: Commit to main

```bash
git add -A && git commit -m "Add <service> MCP server + companion skills"
```

### 7b: Push

Ask the user to push their changes:

"Push your changes to GitHub: `git push origin main`"

Do not push on the user's behalf.

### 7c: Verify CI

Wait for CI to pass. Direct the user to the Actions tab:

"Check your repo's Actions tab to confirm CI passes: `https://github.com/NimbleBrainInc/mcp-<name>/actions`"

### 7d: Cut a release

Guide the user to create a GitHub release:

```bash
gh release create v0.1.0 --title "v0.1.0" --generate-notes
```

### 7e: Verify bundle build

"Check your Actions tab — you should see the **Build MCPB Bundle** workflow running on 3 runners (linux-amd64, linux-arm64, darwin-arm64)."

### 7f: Confirm publication

Once the build completes, confirm the bundle is announced on the registry:

```bash
mpak search <name>
```

## References

See `references/` in this skill for:
- `CONVENTIONS.md` — Naming, manifest format, versioning, build system, entry points (Python + TypeScript)
- `PATTERNS.md` — Complete code patterns, directory structures, CI workflows (Python + TypeScript)
- `SKILL_FORMAT.md` — Skill frontmatter specification and validation rules

Python templates are also available in `templates/` within this skill folder.

For TypeScript, the canonical patterns are in the `NimbleBrainInc/mcp-server-template-typescript` GitHub template repo — already cloned when the project was created.
