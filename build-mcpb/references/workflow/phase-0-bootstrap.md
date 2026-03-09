# Phase 0: Bootstrap

Set up language, service name, repo, and template customization — everything needed before the build pipeline begins.

## 0a: Entry Routing

Determine whether this is a **warm start** or **cold start**:

- **Warm start:** Invoked from `/nimblebrain-contributor` (or after it) in the same session. The conversation already contains a service name, language choice, and the environment has been checked. Heuristic: if the conversation already contains a service name from prior nimblebrain-contributor interaction, treat as warm.
- **Cold start:** Standalone `/build-mcpb` invocation with no prior context. Need to establish everything from scratch.

## 0b: Prerequisites (cold-start only)

If cold start, verify the basics before proceeding:

1. **gh CLI** — `gh auth status` succeeds
2. **Language toolchain** — Python: `uv --version`, `ruff --version`, `ty --version`; TypeScript: `node --version`, `npm --version`
3. **mpak CLI** — `mpak --version` succeeds
4. **GitHub owner** — detect the user's login via `gh api user --jq .login` as a default, then ask: "Repo owner will be `<detected_login>` — or would you prefer a different org/account?" If the user specifies a different owner, use that as `<github_owner>`. The mpak registry allows any GitHub user or org to publish under their own scope (`@<owner>/*` must match the repo owner via OIDC), so this can be a personal account or any org the user has access to.

If any check fails, tell the contributor what's missing and point them to `~/.claude/skills/nimblebrain-contributor/references/DEV_SETUP.md` for setup instructions. Don't block on optional tools (e.g., mpak-scanner) — just note they're unavailable and skip the phases that need them.

## 0c: Language

- **Warm start:** Use the language from the conversation context.
- **Cold start:** Check the current working directory:
  - `pyproject.toml` exists → **Python**
  - `package.json` exists → **TypeScript**
  - Neither → ask the user which language they're using
  - Both → ask the user (unusual — clarify which is primary)

## 0d: Service Name

- **Warm start:** Use the service name from the conversation context.
- **Cold start:**
  1. If `manifest.json` exists, parse the `name` field and strip the scope: `@<scope>/<name>` → `<name>`
  2. Otherwise, derive from the directory name: strip `mcp-` prefix (e.g., `mcp-stripe` → `stripe`)
  3. If neither works, ask the user

## 0e: Naming Variables

Derive these key values from `<name>` (the service name):

| Variable | Description |
|---|---|
| `<display>` (human-readable) | Ask the user, e.g. "JSONPlaceholder" |
| `<github_owner>` | Detected in Phase 0b (user's login by default, or user-specified org) |

For all other naming conventions — module paths, package scopes, env vars, source directories, language-specific derived names — see `references/CONVENTIONS.md`.

## 0f: Repo Creation + Template Customization

**Skip this sub-section** if `manifest.json` already exists with the correct service name — the repo is already set up.

Otherwise:

1. Make sure the user is outside of any existing git repo. If they aren't, help them navigate to a suitable directory first.

2. Confirm before running:
   "This will create `<github_owner>/mcp-<name>` on GitHub as a **public** repo. Want to change the visibility (private/internal), or good to go?"

3. Once confirmed (use `--public` by default, or `--private`/`--internal` per user choice):

   Python:
   ```bash
   gh repo create <github_owner>/mcp-<name> \
     --template NimbleBrainInc/mcp-server-template-python --public --clone
   ```

   Node / TypeScript:
   ```bash
   gh repo create <github_owner>/mcp-<name> \
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
   - `@nimblebraininc/example` → `@<github_owner>/<name>` (registry identifier)
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
   - `YOUR_GITHUB_USERNAME` → `<github_owner>` (already detected in Phase 0b)
   - `YOUR_SERVICE` → `<display>`

After substitutions, do a quick sanity check — grep for any remaining `example`/`Example`/`EXAMPLE` (Python) or `YOUR_` (TypeScript) across the project. If any remain, fix them. Then confirm to the user: "Template customized — all placeholder names replaced with `<name>`."

## 0g: API Key Readiness

Ask the user to have their API key for the target service ready. They'll need it during Phase 3 implementation.

## Gate

Show bootstrap summary and ask the user to confirm before proceeding. Skip this prompt only when warm-start with all values consistent.

```
=> Bootstrap complete:
   Language: [Python/TypeScript]
   Service: <name>
   Module: <module>
   Env var: <NAME>_API_KEY
   Repo: <github_owner>/mcp-<name>

=> Ready to start the build pipeline? [Y/n]
```

**Criteria:**
- [ ] Language chosen (Python or TypeScript)
- [ ] Service name derived, `<display>` and `<github_owner>` confirmed
- [ ] Repo created from template with all placeholders replaced (or existing repo validated)
- [ ] Contributor has confirmed bootstrap summary

**If any criterion fails:** Revisit the relevant sub-step above.

**When all pass:** Proceed to Phase 1.
