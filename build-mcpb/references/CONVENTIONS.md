# MCP Server Conventions

## Critical

Package names MUST be scoped: `@nimblebraininc/<name>`

Registry rejects other formats (e.g., `ai.nimbletools/name`).

## Python Server Manifest

Servers using src-layout MUST use module execution:
```json
{
  "server": {
    "type": "python",
    "entry_point": "mcp_module.server",
    "mcp_config": {
      "command": "python",
      "args": ["-m", "mcp_module.server"]
    }
  }
}
```

Never use file paths (`${__dirname}/...`) for Python, as it breaks after bundling.

Python servers with src-layout MUST have a `[build-system]` in `pyproject.toml`:
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/mcp_<name>"]
```
Without this, `uv sync` won't install the package and module imports silently fail.

## user_config (API Keys)

Required format for `mpak config set` compatibility:
```json
{
  "user_config": {
    "api_key": {
      "type": "string",
      "title": "Human Readable Title",
      "description": "Where to get this key",
      "sensitive": true,
      "required": true
    }
  },
  "server": {
    "mcp_config": {
      "env": {
        "SERVICE_API_KEY": "${user_config.api_key}"
      }
    }
  }
}
```

Key points:
- Field names: lowercase with underscores (`api_key`, `database_uri`)
- Use `sensitive: true` for secrets (not `secret`)
- Reference via `${user_config.<field_name>}` in env mapping

## Python Server Entry Points

Python servers need two entrypoints for dual transport:

```python
# ASGI entrypoint (container deployment)
app = mcp.http_app()

# Stdio entrypoint (mpak / Claude Desktop)
if __name__ == "__main__":
    mcp.run()
```

| Context | Command | Entrypoint |
|---------|---------|------------|
| Containers | `uvicorn module.server:app` | `app = mcp.http_app()` |
| mpak / Claude Desktop | `python -m module.server` | `__main__` block |
| Local dev (HTTP) | `uvicorn module.server:app` | `app = mcp.http_app()` |
| Local dev (stdio) | `python -m module.server` | `__main__` block |

Both entrypoints are required. The `app` object must exist at module level for uvicorn to import it.

## Build Workflow Base

Uses mcpb-pack v2 with release trigger:
```yaml
on:
  release:
    types: [published]

permissions:
  contents: write
  id-token: write

jobs:
  build:
    strategy:
      matrix:
        include:
          - os: linux
            arch: amd64
            runner: ubuntu-latest
          - os: linux
            arch: arm64
            runner: ubuntu-24.04-arm
          - os: darwin
            arch: arm64
            runner: macos-latest
    runs-on: ${{ matrix.runner }}
```

## Version Management

Version lives in four files that MUST stay in sync:

| File | Field | Purpose |
|------|-------|---------|
| `manifest.json` | `version` | MCPB bundle version (what mpak sees) |
| `server.json` | `version` | NimbleBrain registry metadata |
| `pyproject.toml` | `version` | Python package version |
| `src/<package>/__init__.py` | `__version__` | Runtime version |

Bump all at once: `make bump VERSION=0.2.0`

## Versioning Policy

All NimbleBrain MCP servers follow Semantic Versioning 2.0.0 with a deliberate choice to remain in the v0.x.y range until the MCP ecosystem stabilizes.

In v0.x.y:
- **x** (minor): Incremented for any notable change, including breaking changes to tool definitions
- **y** (patch): Incremented for bug fixes and minor improvements that don't change the tool surface

A server may graduate to v1.0.0 when ALL of the following are true:
1. The MCP protocol has reached 1.0 or demonstrated long-term stability
2. The server's tool surface has been stable for 3+ months
3. There is meaningful external adoption
4. The maintainers are confident the tool definitions are "correct"

## Release Process

```bash
# 1. Bump version
make bump VERSION=0.2.0

# 2. Commit the version bump
git add -A && git commit -m "Bump version to 0.2.0"

# 3. Tag and push
git tag v0.2.0 && git push origin main v0.2.0

# 4. Create GitHub release (triggers CI bundle build)
gh release create v0.2.0 --title "v0.2.0" --notes "- changelog"
```

## Tooling

- **Package manager**: uv (not pip)
- **Linting/formatting**: ruff (not flake8, black, isort)
- **Type checking**: ty (not mypy, pyright)
- **Testing**: pytest with pytest-asyncio
