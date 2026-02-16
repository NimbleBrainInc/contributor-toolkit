# MCP Server Code Patterns

Complete code patterns for building Python/FastMCP MCP servers.

## Directory Structure

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
├── .mcpbignore             # Excludes for bundle
├── CLAUDE.md               # Dev docs
├── Makefile
├── manifest.json           # MCPB manifest (v0.4)
├── pyproject.toml          # uv + ruff + ty
├── pytest.ini
└── README.md
```

## manifest.json (MCPB v0.4)

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
        "API_KEY": "${user_config.api_key}"
      }
    }
  }
}
```

## pyproject.toml

```toml
[project]
name = "mcp-<name>"
version = "0.1.0"
description = "..."
requires-python = ">=3.13"
dependencies = [
    "aiohttp>=3.11.0",
    "fastmcp>=2.14.0",
    "pydantic>=2.0.0",
    "python-dotenv>=1.0.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/mcp_<name>"]

[tool.ruff]
target-version = "py313"
line-length = 100

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "C4", "UP"]
ignore = ["E501", "B008"]

[dependency-groups]
dev = [
    "pytest>=8.4.0",
    "pytest-asyncio>=0.24.0",
    "ruff>=0.13.0",
    "ty>=0.1.0",
]
```

## server.py Pattern

```python
"""<Service> MCP Server - FastMCP Implementation."""

import logging
import os
import sys

from fastmcp import Context, FastMCP
from starlette.requests import Request
from starlette.responses import JSONResponse

from mcp_<name>.api_client import Client, APIError
from mcp_<name>.api_models import SomeModel

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    stream=sys.stderr,
)
logger = logging.getLogger("mcp_<name>")

mcp = FastMCP("<Service>")

_client: Client | None = None


def get_client(ctx: Context | None = None) -> Client:
    """Get or create the API client instance."""
    global _client
    if _client is None:
        api_key = os.environ.get("<NAME>_API_KEY")
        if not api_key:
            msg = "<NAME>_API_KEY environment variable is required"
            if ctx:
                ctx.error(msg)
            raise ValueError(msg)
        _client = Client(api_key=api_key)
    return _client


@mcp.custom_route("/health", methods=["GET"])
async def health_check(request: Request) -> JSONResponse:
    """Health check endpoint for monitoring."""
    return JSONResponse({"status": "healthy", "service": "mcp-<name>"})


@mcp.tool()
async def some_tool(arg: str, ctx: Context | None = None) -> SomeModel:
    """Tool description.

    Args:
        arg: Argument description
        ctx: MCP context

    Returns:
        Return description
    """
    client = get_client(ctx)
    try:
        return await client.some_method(arg)
    except APIError as e:
        if ctx:
            ctx.error(f"API error: {e.message}")
        raise


# ASGI app for HTTP deployment
app = mcp.http_app()

# Stdio entrypoint for Claude Desktop / mpak
if __name__ == "__main__":
    mcp.run()
```

## api_client.py Pattern

```python
import os
from typing import Any
import aiohttp
from aiohttp import ClientError

from .api_models import SomeModel, SomeListResponse


class APIError(Exception):
    def __init__(self, status: int, message: str, details: dict[str, Any] | None = None) -> None:
        self.status = status
        self.message = message
        self.details = details
        super().__init__(f"API Error {status}: {message}")


class Client:
    BASE_URL = "https://api.example.com/v1"

    def __init__(self, api_key: str | None = None, timeout: float = 30.0) -> None:
        self.api_key = api_key or os.environ.get("API_KEY")
        if not self.api_key:
            raise ValueError("API_KEY is required")
        self.timeout = timeout
        self._session: aiohttp.ClientSession | None = None

    async def _ensure_session(self) -> None:
        if not self._session:
            headers = {
                "Authorization": f"Bearer {self.api_key}",
                "Accept": "application/json",
                "Content-Type": "application/json",
            }
            self._session = aiohttp.ClientSession(
                headers=headers,
                timeout=aiohttp.ClientTimeout(total=self.timeout)
            )

    async def close(self) -> None:
        if self._session:
            await self._session.close()
            self._session = None

    async def _request(
        self,
        method: str,
        path: str,
        params: dict[str, Any] | None = None,
        json_data: Any | None = None,
    ) -> dict[str, Any]:
        await self._ensure_session()
        url = f"{self.BASE_URL}{path}"

        if params:
            params = {k: v for k, v in params.items() if v is not None}

        try:
            if not self._session:
                raise RuntimeError("Session not initialized")

            kwargs: dict[str, Any] = {}
            if json_data is not None:
                kwargs["json"] = json_data
            if params:
                kwargs["params"] = params

            async with self._session.request(method, url, **kwargs) as response:
                result = await response.json()

                if response.status >= 400:
                    error_msg = result.get("message", "Unknown error")
                    raise APIError(response.status, error_msg, result)

                return result
        except ClientError as e:
            raise APIError(500, f"Network error: {str(e)}") from e

    async def list_items(self, limit: int = 20) -> list[SomeModel]:
        data = await self._request("GET", "/items", params={"limit": limit})
        response = SomeListResponse(**data)
        return response.data.items
```

## api_models.py Pattern

```python
from enum import Enum
from typing import Any
from pydantic import BaseModel, Field


class SomeEnum(str, Enum):
    VALUE_A = "value_a"
    VALUE_B = "value_b"


class Pagination(BaseModel):
    model_config = {"populate_by_name": True}
    next_link: str | None = Field(default=None, alias="nextLink")


class SomeModel(BaseModel):
    id: str = Field(..., description="ID")
    name: str | None = Field(None, description="Name")
    created_at: str | None = Field(None, alias="createdAt")
    custom_fields: dict[str, Any] = Field(default_factory=dict, alias="customFieldValues")


class SomeListResponse(BaseModel):
    class Data(BaseModel):
        items: list[SomeModel] = Field(default_factory=list)
        pagination: Pagination = Field(default_factory=lambda: Pagination())
    data: Data
```

## .mcpbignore

```
.venv/
.git/
.github/
.pytest_cache/
__pycache__/
*.pyc
.ruff_cache/
.coverage
tests/
dist/
build/
*.egg-info/
*.mcpb
Makefile
pytest.ini
.env
.env.*
uv.lock
.vscode/
.idea/
CLAUDE.md
*.png
```

## GitHub Workflows

### build-bundle.yml
```yaml
name: Build MCPB Bundle

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
    steps:
      - uses: actions/checkout@v4
      - name: Update manifest version
        run: |
          VERSION="${{ github.event.release.tag_name }}"
          VERSION="${VERSION#v}"
          jq --arg v "$VERSION" '.version = $v' manifest.json > manifest.tmp.json
          mv manifest.tmp.json manifest.json
      - uses: NimbleBrainInc/mcpb-pack@v2
        with:
          output: "{name}-{version}-${{ matrix.os }}-${{ matrix.arch }}.mcpb"
```

### ci.yml
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv python install 3.13
      - run: uv sync --dev
      - run: uv run ruff format --check src/ tests/
      - run: uv run ruff check src/ tests/
      - run: uv run ty check src/

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv python install 3.13
      - run: uv sync --dev
      - run: uv run pytest tests/ -v
```

## Build & Test Commands

```bash
uv sync --dev                        # Install dependencies
uv run ruff format src/ tests/       # Format
uv run ruff check src/ tests/        # Lint
uv run ty check src/                 # Type check
uv run pytest tests/ -v              # Test
make check                           # All of the above

# Test locally
API_KEY=xxx uv run python -m mcp_<name>.server

# Test with MCP Inspector
API_KEY=xxx npx @modelcontextprotocol/inspector uv run python -m mcp_<name>.server
```

## Tool Design Guidelines

1. **One tool per operation**: `list_items`, `get_item`, `create_item`, etc.
2. **Clear docstrings**: Args, returns, description
3. **Sensible defaults**: pagination limits, etc.
4. **Error handling**: Catch API errors, log via context
5. **Search tools**: Add convenience search wrappers if API supports filtering

## API Analysis Checklist

When analyzing an API to build an MCP server:

1. **Authentication**: Bearer token, API key, OAuth?
2. **Base URL**: What's the API base URL?
3. **Resources**: What entities exist? (people, companies, etc.)
4. **Operations**: CRUD operations available?
5. **Pagination**: Cursor-based, offset-based?
6. **Filtering**: What filter parameters are supported?
7. **Response format**: JSON structure, nested objects?
8. **Rate limits**: Any limits to be aware of?
9. **OpenAPI spec**: Is one available for type generation?
