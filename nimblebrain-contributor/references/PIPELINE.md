# MCP Server Build Pipeline

## Overview

```
Scaffold → Implement → Verify → Validate Bundle → Author Skills → Release
```

All phases are handled by `/build-mcpb`. The skill auto-detects language and service name from the repo.

## Step 1: Scaffold

```
> /build-mcpb
```

The template repo already provides the full project structure:

**Python:**
- `src/mcp_<name>/server.py` - FastMCP server with tool stubs
- `src/mcp_<name>/api_client.py` - Async HTTP client
- `src/mcp_<name>/api_models.py` - Pydantic response models
- `manifest.json` - MCPB v0.4 manifest
- `Makefile` - Build/test/clean/package targets
- `.github/workflows/` - CI and release workflows
- `tests/` - Test stubs
- `README.md` - Documentation template

**TypeScript:**
- `src/index.ts` - MCP server entry point with tool registration
- `src/handlers/` - One file per tool
- `src/utils/apiClient.ts` - HTTP client
- `src/types.ts` - Zod API response schemas
- `src/schemas.ts` - Zod tool input schemas
- `src/formatters.ts` - Response formatters
- `src/config.ts` - Environment variable configuration
- `manifest.json` - MCPB v0.4 manifest
- `server.json` - Registry metadata
- `package.json`, `tsconfig.json`, `Makefile`
- `.github/workflows/` - CI and release workflows
- `tests/` - Test stubs
- `README.md` - Documentation template

## Step 2: Implement Tools

**Python:** Fill in the tool implementations in `server.py`. Each tool should:
- Have a clear docstring with Args/Returns
- Use the async client for API calls
- Return typed Pydantic models
- Handle errors via context logging

**TypeScript:** Implement one handler per tool in `src/handlers/`. Each tool should:
- Have a Zod input schema in `schemas.ts`
- Use the API client for API calls
- Return formatted responses via `formatters.ts`
- Handle errors via `errorResponse()`

## Step 3: Local Verification

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

## Step 4: Validate Bundle

Phase 5 of `/build-mcpb` handles manifest validation, build verification, bundle inspection, MTF compliance (if scanner is available), and runtime validation with a full MCP handshake.

```bash
make bundle
```

## Step 5: Author Companion Skills

Phase 6 of `/build-mcpb` analyzes your tools, suggests 3-5 workflow skills, and generates SKILL.md + references for each.

## Step 6: Validate and Pack Skills

```bash
mpak skill validate ./skills/<skill-name>
mpak skill pack ./skills/<skill-name>
```

## Step 7: Release

The contributor owns the repo — there is no PR step. The release flow is:

1. Commit all changes to `main`
2. Push to GitHub
3. Wait for CI to pass
4. Cut a GitHub release:
   ```bash
   gh release create v0.1.0 --title "v0.1.0" --generate-notes
   ```
5. The `build-bundle.yml` workflow triggers automatically and builds bundles on 3 runners (linux-amd64, linux-arm64, darwin-arm64)
6. Once complete, the bundle is announced on the mpak registry:
   ```bash
   mpak search <name>
   ```

## Version Management

Version lives in files that MUST stay in sync:

**Python:**

| File | Field |
|------|-------|
| `manifest.json` | `version` |
| `server.json` | `version` |
| `pyproject.toml` | `version` |
| `src/<package>/__init__.py` | `__version__` |

**TypeScript:**

| File | Field |
|------|-------|
| `manifest.json` | `version` |
| `server.json` | `version` |
| `package.json` | `version` |
| `src/constants.ts` | `SERVER_VERSION` (via `make sync`) |

Bump all at once: `make bump VERSION=0.2.0`

## Release Process

```bash
make bump VERSION=0.2.0
git add -A && git commit -m "Bump version to 0.2.0"
git tag v0.2.0 && git push origin main v0.2.0
gh release create v0.2.0 --title "v0.2.0" --notes "- changelog"
```

GitHub Actions builds and uploads MCPB bundles automatically on release.
