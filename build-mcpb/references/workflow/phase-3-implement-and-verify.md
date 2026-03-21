# Phase 3: Implement & Verify

Write tool logic, models, and client code, then run all checks to ensure the implementation is correct before bundling.

## Concept Mapping

| Concept | Python | TypeScript |
|---------|--------|------------|
| Response models | Pydantic `BaseModel` | Zod `z.object()` in `types.ts` |
| Tool input validation | Python type hints + FastMCP | Zod schemas in `schemas.ts` |
| HTTP client | `aiohttp.ClientSession` | `fetch()` built-in |
| Tool registration | `@mcp.tool()` decorator | `server.registerTool()` call |
| Response formatting | Return Pydantic model directly | `formatters.ts` → JSON.stringify |
| Error handling | `try/except APIError` | `try/catch` → `errorResponse()` |

## If Python

Implement in this order:
1. **`api_models.py`** — Pydantic models for API responses. Use `Field(alias=...)` for camelCase mapping.
2. **`api_client.py`** — Async aiohttp client. Set BASE_URL, add one method per endpoint.
3. **`server.py`** — FastMCP server with `@mcp.tool()` decorators. Global client with lazy init. Dual transport (http_app + stdio).
4. **`manifest.json`** + **`server.json`** — Fill all placeholder fields. See `references/CONVENTIONS.md` for the full `server.json` schema.

See `references/PATTERNS.md` → "Python Server Patterns" for complete code examples.

## If TypeScript

Implement in this order:
1. **`src/types.ts`** — Zod schemas for API response shapes
2. **`src/utils/apiClient.ts`** — Rename class, set BASE_URL, add methods. Update import in `errorResponse.ts`.
3. **`src/schemas.ts`** — Zod input schema for each tool
4. **`src/formatters.ts`** — One formatter per resource type
5. **`src/handlers/<tool>.ts`** — One file per tool
6. **`src/index.ts`** — `server.registerTool()` for each tool
7. **`src/config.ts`** — Rename env var
8. **`manifest.json`** + **`server.json`** — Fill all TODO fields, then `make sync`

See `references/PATTERNS.md` → "TypeScript Server Patterns" for complete code examples.

## Critical Notes (TypeScript)

- Use **exact dependency versions** (no `^` or `~`) — range specifiers are L2 MTF findings
- Use `.js` extensions in all imports (Node ESM requirement)
- Never edit `src/constants.ts` manually — use `make sync`
- Never edit `.github/workflows/` — shared infrastructure
- The embedded SKILL.md lives in `src/` and is copied to `build/` during compilation

## Verify

Run all checks and fix any issues before proceeding.

### Unit tests and code quality

**Python:**
```bash
uv sync --dev
make check                    # format, lint, typecheck, unit tests
```

Mocked HTTP, FastMCP Client-based tool tests, skill resource tests. The template scaffolds these — fill them in for every tool. `make check` must pass before proceeding.

See `references/PATTERNS.md` → "Test Patterns (Python)" for the FastMCP Client pattern, ToolError handling, and mock fixtures.

**TypeScript:**
```bash
make check
```
This runs: `format:check` → `lint` → `typecheck` → `test`

### Integration tests

**Python:**
```bash
make test-integration         # real API calls (needs <NAME>_API_KEY in .env)
```

Real API calls against the live service. The template scaffolds `tests-integration/test_core_tools.py` as a stub — you must replace it with real tests.

**How to write them:** Open `api_client.py` and list every public method (skip `__init__`, `close`, `_request`, `_ensure_session`, and dunder methods). For each method, write a test:

- **Read methods** (list, get, search): Call with minimal valid parameters, assert the response has the expected shape (list, dict, or model with expected keys).
- **Write methods** (create, update, delete): Create a test resource, verify it, then clean it up in a `finally` block. If no delete method exists, mark the resource as completed/archived and leave a comment.
- **Chained methods:** Some methods need an ID from a prior call (e.g., `list_workspaces` returns a GID needed by `search_tasks`). Chain them — call the list method first, use the first result's ID.
- **Tier-gated methods:** If the API has premium endpoints that may not be available on the user's plan, write a `has_<feature>_access` helper that probes the endpoint and returns `False` on 401/402/403. Use `pytest.skip()` in the test if access is unavailable.

See `references/PATTERNS.md` → "Integration Test Patterns (Python)" for concrete examples.

**How to run them:**

1. Ask the contributor to add their API key to `.env` (e.g., `ASANA_API_KEY=xxx`). The `.env` file is already in `.gitignore` and `.mcpbignore`. The contributor was asked for this key at the start of the process — they should have it ready.
2. Run `make test-integration`.
3. All tests should pass or skip (for tier-gated features). Fix any failures before proceeding.
4. If the contributor says auth setup is too complex for now (e.g., OAuth flows, multi-step app configuration), proceed — the tests are written and ready to run later. Do not skip writing the tests.

### LLM smoke tests

**Python:**
```bash
make test-llm                 # needs <NAME>_API_KEY + ANTHROPIC_API_KEY in .env
```

Verify Claude Haiku selects the correct tool given the skill resource. Requires both the service API key and `ANTHROPIC_API_KEY`.

**How to write them:** The template scaffolds `get_server_context()` and `get_anthropic_client()` — leave those as-is. Replace the commented-out test stub with 3–5 real tests, one per key tool. Extract a `call_llm()` helper to avoid repeating the system prompt construction across tests.

Each test sends a natural language prompt and asserts the LLM selected the expected tool. Include concrete values for any required parameters in the prompt (IDs, coordinates, dates) — without them, the LLM will ask for clarification instead of calling the tool.

See `references/PATTERNS.md` → "LLM smoke tests" for the `call_llm()` helper pattern and the concrete-identifiers rule.

**How to run them:**

1. Ask the contributor to add `ANTHROPIC_API_KEY` to `.env` alongside the service API key.
2. Run `make test-llm`.
3. All tests should pass. If a test fails because the LLM picked the wrong tool, adjust the prompt to be more specific before touching the SKILL.md.
4. If the contributor does not have an `ANTHROPIC_API_KEY`, proceed — the tests are written and ready to run later. Do not skip writing the tests.

## Gate

**Criteria:**
- [ ] All tool logic, models, and client code implemented
- [ ] Linting passes with no errors
- [ ] Type checking passes with no errors
- [ ] Unit tests pass (`make check`)
- [ ] Integration tests written with real assertions (not stubs or TODOs)
- [ ] Integration tests pass or skip, if API key is available
- [ ] LLM smoke tests written with real assertions (not stubs or TODOs)
- [ ] LLM smoke tests pass, if `ANTHROPIC_API_KEY` is available

**If any criterion fails:** Fix the reported issues and re-run checks.

**When all pass:** Proceed to Phase 4.
