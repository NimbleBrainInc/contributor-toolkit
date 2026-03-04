---
name: build-mcpb
description: Build MCP servers end-to-end. Scaffolds a production-ready Python or TypeScript server from API documentation, implements tools, validates the MCPB bundle, authors companion skills, and guides release to the mpak registry. Covers the full lifecycle from API analysis to published bundle. Use when building a new MCP server, wrapping an API, or creating an integration. Triggers include "build an MCP server", "create a server for X", "/build-mcpb".
license: Apache-2.0
compatibility: Python 3.13+, uv, ruff, ty OR Node.js 24+, npm. Docker, mpak CLI. Claude Code or Codex with filesystem access.
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
    - codex
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

The skill bootstraps everything needed: language, service name, repo creation, template setup — then proceeds through the build pipeline.

## Pipeline Overview

```
Phase 0: Bootstrap        Language, service name, repo creation, template setup
Phase 1: API Analysis     Fetch docs, identify resources, propose tools
Phase 2: Scaffold         Verify project structure from template
Phase 3: Implement        Write tool logic, models, client
Phase 4: Verify           Lint, typecheck, test
Phase 5: Validate Bundle  Manifest, build, bundle, MTF scan, runtime
Phase 6: Author Skills    Generate 2-3 companion skills
Phase 7: Release          Commit, push, cut release, verify bundle
```

## Phase 0: Bootstrap

Set up language, service name, repo, and template customization — everything needed before the build pipeline begins.

### 0a: Entry Routing

Determine whether this is a **warm start** or **cold start**:

- **Warm start:** Invoked from `/nimblebrain-contributor` (or after it) in the same session. The conversation already contains a service name, language choice, and the environment has been checked. Heuristic: if the conversation already contains a service name from prior nimblebrain-contributor interaction, treat as warm.
- **Cold start:** Standalone `/build-mcpb` invocation with no prior context. Need to establish everything from scratch.

### 0b: Prerequisites (cold-start only)

If cold start, verify the basics before proceeding:

1. **gh CLI** — `gh auth status` succeeds
2. **Language toolchain** — Python: `uv --version`, `ruff --version`, `ty --version`; TypeScript: `node --version`, `npm --version`
3. **mpak CLI** — `mpak --version` succeeds

If any check fails, tell the contributor what's missing and point them to `~/.claude/skills/nimblebrain-contributor/references/DEV_SETUP.md` for setup instructions. Don't block on optional tools (e.g., mpak-scanner) — just note they're unavailable and skip the phases that need them.

### 0c: Language

- **Warm start:** Use the language from the conversation context.
- **Cold start:** Check the current working directory:
  - `pyproject.toml` exists → **Python**
  - `package.json` exists → **TypeScript**
  - Neither → ask the user which language they're using
  - Both → ask the user (unusual — clarify which is primary)

### 0d: Service Name

- **Warm start:** Use the service name from the conversation context.
- **Cold start:**
  1. If `manifest.json` exists, parse the `name` field and strip the scope: `@nimblebraininc/<name>` → `<name>`
  2. Otherwise, derive from the directory name: strip `mcp-` prefix (e.g., `mcp-stripe` → `stripe`)
  3. If neither works, ask the user

### 0e: Naming Variables

Derive these values from `<name>` (the service name):

| Variable | Example (`<name>` = `jsonplaceholder`) |
|---|---|
| `<name>` | `jsonplaceholder` |
| `<Name>` (PascalCase) | `Jsonplaceholder` |
| `<NAME>` (UPPER_SNAKE) | `JSONPLACEHOLDER` |
| `<display>` (human-readable) | Ask the user, e.g. "JSONPlaceholder" |

Language-specific derived names:

| Variable | Python | TypeScript |
|----------|--------|------------|
| Package scope | `@nimblebraininc/<name>` | `@nimblebraininc/<name>` |
| Module / package | `mcp_<name>` (underscores) | `mcp-<name>` (hyphens) |
| Source directory | `src/mcp_<name>/` | `src/` |
| Env var | `<NAME>_API_KEY` | `<NAME>_API_KEY` |

### 0f: Repo Creation + Template Customization

**Skip this sub-section** if `manifest.json` already exists with the correct service name — the repo is already set up.

Otherwise:

1. Make sure the user is outside of any existing git repo. If they aren't, help them navigate to a suitable directory first.

2. Confirm before running:
   "This will create a public repo at `NimbleBrainInc/mcp-<name>` on GitHub. Ready to go?"

3. Once confirmed:

   Python:
   ```bash
   gh repo create NimbleBrainInc/mcp-<name> \
     --template NimbleBrainInc/mcp-server-template-python --public --clone
   ```

   Node / TypeScript:
   ```bash
   gh repo create NimbleBrainInc/mcp-<name> \
     --template NimbleBrainInc/mcp-server-template-typescript --public --clone
   ```

4. **Immediately after cloning, `cd` into `mcp-<name>` and replace all template placeholders with the actual service name.** Do not move on until this is done — downstream phases assume the project already has correct names everywhere.

**Python template substitutions:**

1. Rename the package directory:
   ```bash
   mv src/mcp_example src/mcp_<name>
   ```
2. Replace across all files (`*.py`, `*.toml`, `*.json`, `*.md`, `Makefile`, `.env.example`):
   - `mcp_example` → `mcp_<name>` (package name in imports, paths, logger)
   - `mcp-example` → `mcp-<name>` (project/bundle name)
   - `@nimblebraininc/example` → `@nimblebraininc/<name>` (registry identifier)
   - `ExampleClient` → `<Name>Client` (class name)
   - `ExampleAPIError` → `<Name>APIError` (class name)
   - `EXAMPLE_API_KEY` → `<NAME>_API_KEY` (env var)
   - `https://api.example.com/v1` → leave as TODO for Phase 3 to fill with actual API URL
   - `https://example.com/settings/api` → leave as TODO for Phase 3
   - `mcp-server-example` → `mcp-server-<name>` (User-Agent)
   - `FastMCP("Example")` → `FastMCP("<display>")` (server display name)
   - `"example"` in `pyproject.toml` keywords → `"<name>"`
   - Update `pyproject.toml` URLs to use `mcp-<name>` repo name
   - Update README title and description to reference `<display>` instead of "Example"

**TypeScript template substitutions:**

Replace across all files (`*.ts`, `*.json`, `*.md`, `Makefile`, `CLAUDE.md`):
   - `YOUR_SERVER_NAME` → `<name>`
   - `YOUR_DISPLAY_NAME` → `<display>`
   - `YOUR_REPO_NAME` → `mcp-<name>`
   - `YOUR_API_KEY_ENV_VAR` → `<NAME>_API_KEY`
   - `YOUR_API_HOST` → leave as TODO for Phase 3
   - `YOUR_API_BASE_URL` → leave as TODO for Phase 3
   - `YOUR_GITHUB_USERNAME` → look up via `gh api user -q .login`
   - `YOUR_SERVICE` → `<display>`

After substitutions, do a quick sanity check — grep for any remaining `example`/`Example`/`EXAMPLE` (Python) or `YOUR_` (TypeScript) across the project. If any remain, fix them. Then confirm to the user: "Template customized — all placeholder names replaced with `<name>`."

### 0g: API Key Readiness

Ask the user to have their API key for the target service ready. They'll need it during Phase 3 implementation.

### 0h: Confirm

Show bootstrap summary and ask the user to confirm before proceeding. Skip this prompt only when warm-start with all values consistent.

```
=> Bootstrap complete:
   Language: [Python/TypeScript]
   Service: <name>
   Module: <module>
   Env var: <NAME>_API_KEY
   Repo: NimbleBrainInc/mcp-<name>

=> Ready to start the build pipeline? [Y/n]
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

Phase 0 created the repo from template and customized all placeholders. Verify it's intact.

**Python** — check that these exist:
- `src/mcp_<name>/server.py`, `api_client.py`, `api_models.py`, `__init__.py`
- `tests/`
- `manifest.json`, `server.json`, `pyproject.toml`, `Makefile`
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
4. **`manifest.json`** + **`server.json`** — Fill all placeholder fields. See `references/CONVENTIONS.md` for the full `server.json` schema.

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
- Skills created must be **self-contained** in a single `SKILL.md` (no `references/` subdirectory)

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

### 7a: Pre-flight Checks

Before committing, verify the release will succeed:

1. **Version consistency** — Read the version from `manifest.json` and confirm it matches across all version sources:
   - Python: `pyproject.toml`, `server.json`, `src/mcp_<name>/__init__.py`
   - TypeScript: run `make sync` and verify no diff
   If anything is out of sync, run `make bump VERSION=<version>` to fix.
2. **`mpak.json`** — Must exist in the repo root with `name` matching `manifest.json`. Without it, the registry announce fails silently after the bundle builds.
3. **No secrets in working tree** — Check for `.env` files or anything containing real API keys. Warn the contributor if found; do not stage them.
4. **Skills validated** — For each skill in `skills/`, run `mpak skill validate ./skills/<skill-name>` and fix any issues before committing.

### 7b: Stage and Commit

Do not use `git add -A`. Review first, then stage explicitly:

1. Run `git status` and show the contributor what will be committed.
2. Stage server source, tests, skills, manifests, config, and workflow files. Do NOT stage `.env`, credentials, or debug artifacts.
3. Derive the version from `manifest.json` for the commit message:

```bash
VERSION=$(jq -r .version manifest.json)
git add <files...>
git commit -m "Add <service> MCP server v${VERSION} + companion skills"
```

### 7c: Push

Do NOT push on the contributor's behalf. Ask them:

"Push your changes to GitHub: `git push origin main`"

### 7d: Verify CI

After the push, actively monitor CI — don't just point to a URL:

```bash
gh run list --repo NimbleBrainInc/mcp-<name> --branch main --limit 1
gh run watch --repo NimbleBrainInc/mcp-<name>
```

If CI fails: read the logs with `gh run view <run-id> --log-failed`, help the contributor diagnose and fix, then create a new commit (not amend).

### 7e: Cut Release

Once CI passes, derive the tag from `manifest.json`:

```bash
VERSION=$(jq -r .version manifest.json)
gh release create "v${VERSION}" --title "v${VERSION}" --generate-notes
```

### 7f: Monitor Bundle Build

The release triggers `build-bundle.yml` on 3 runners (linux-amd64, linux-arm64, darwin-arm64). Track it:

```bash
gh run list --repo NimbleBrainInc/mcp-<name> --workflow=build-bundle.yml --limit 1
gh run watch --repo NimbleBrainInc/mcp-<name>
```

If a runner fails, check logs with `gh run view <run-id> --log-failed`. Common issues:
- **Dependency vendor fails** — version conflicts in `pyproject.toml` / `package.json`
- **mcpb pack fails** — `manifest.json` format issues (re-check against Phase 5a checklist)
- **OIDC announce fails** — `mpak.json` missing or `name` doesn't match manifest

### 7g: Wrap Up

Once all 3 runners succeed, the server bundle is announced to the mpak registry. Tell the contributor:

"Your `<display>` MCP server is built, bundled, and announced to the mpak registry. It may take several minutes to propagate. Once live, anyone can run it:

```
mpak run @nimblebraininc/<name>
```

To check availability:
```bash
mpak search <name>
```

**Note on companion skills:** The skills in `skills/` are committed to your repo but are not included in the `.mcpb` server bundle — skills are distributed separately. Skill publishing from server repos is not yet automated; this will be addressed in a future template update."

## References

See `references/` in this skill for:
- `CONVENTIONS.md` — Naming, manifest format, versioning, build system, entry points (Python + TypeScript)
- `PATTERNS.md` — Complete code patterns, directory structures, CI workflows (Python + TypeScript)
- `SKILL_FORMAT.md` — Skill frontmatter specification and validation rules

The canonical project structure comes from the GitHub template repos (`NimbleBrainInc/mcp-server-template-python` and `NimbleBrainInc/mcp-server-template-typescript`), cloned and customized during Phase 0 Bootstrap.
