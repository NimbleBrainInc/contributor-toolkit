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

**TypeScript:**
```bash
make check
```
This runs: `format:check` → `lint` → `typecheck` → `test`

## Gate

**Criteria:**
- [ ] All tool logic, models, and client code implemented
- [ ] Linting passes with no errors
- [ ] Type checking passes with no errors
- [ ] All tests pass

**If any criterion fails:** Fix the reported issues and re-run checks.

**When all pass:** Proceed to Phase 4.
