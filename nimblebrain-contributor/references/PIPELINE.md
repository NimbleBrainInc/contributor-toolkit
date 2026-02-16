# MCP Server Build Pipeline

## Overview

```
Scaffold → Implement → Verify → Validate → Author Skills → Pack → PR
```

## Step 1: Scaffold with /build-mcp

```
> "Build me an MCP server for the Notion API"
```

This creates the full project structure:
- `src/mcp_<name>/server.py` - FastMCP server with tool stubs
- `src/mcp_<name>/api_client.py` - Async HTTP client
- `src/mcp_<name>/api_models.py` - Pydantic response models
- `manifest.json` - MCPB v0.4 manifest
- `Makefile` - Build/test/clean/package targets
- `.github/workflows/` - CI and release workflows
- `tests/` - Test stubs
- `README.md` - Documentation template

## Step 2: Implement Tools

Fill in the tool implementations in `server.py`. Each tool should:
- Have a clear docstring with Args/Returns
- Use the async client for API calls
- Return typed Pydantic models
- Handle errors via context logging

## Step 3: Local Verification

```bash
uv sync --dev
uv run ruff format src/ tests/
uv run ruff check src/ tests/
uv run ty check src/
uv run pytest tests/ -v
```

## Step 4: Validate with /validate-mcpb

```
> "Validate this bundle"
```

Runs 6 phases: manifest, build, bundle inspection, MTF scan, runtime, registry compatibility.

## Step 5: Author Skills with /author-skills-for-server

```
> "Write skills for this server"
```

Analyzes your tools, suggests 3-5 workflow skills, generates SKILL.md + references for each.

## Step 6: Validate and Pack Skills

```bash
mpak skill validate ./skills/<skill-name>
mpak skill pack ./skills/<skill-name>
```

## Step 7: Submit PR

Include both server code and companion skills. CI must pass before review.

## Version Management

Version lives in four files that MUST stay in sync:

| File | Field |
|------|-------|
| `manifest.json` | `version` |
| `server.json` | `version` |
| `pyproject.toml` | `version` |
| `src/<package>/__init__.py` | `__version__` |

Bump all at once: `make bump VERSION=0.2.0`

## Release Process

```bash
make bump VERSION=0.2.0
git add -A && git commit -m "Bump version to 0.2.0"
git tag v0.2.0 && git push origin main v0.2.0
gh release create v0.2.0 --title "v0.2.0" --notes "- changelog"
```

GitHub Actions builds and uploads MCPB bundles automatically on release.
