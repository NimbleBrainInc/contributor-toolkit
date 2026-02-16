# Developer Environment Setup

## Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Python | 3.13+ | Server runtime |
| uv | latest | Python package manager (replaces pip) |
| ruff | latest | Linting and formatting (replaces flake8/black/isort) |
| ty | latest | Type checking (replaces mypy/pyright) |
| Docker | latest | Testing MCPB bundles locally |
| gh | latest | GitHub CLI for repo creation and PRs |
| mpak | latest | MCP bundle CLI (search, validate, pack, install) |
| Node.js | 20+ | For mpak CLI (npm package) |

## macOS Setup (Recommended)

```bash
# 1. Install uv (Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. Install Python 3.13
uv python install 3.13

# 3. Install development tools
uv tool install ruff
uv tool install ty

# 4. Install Docker
brew install --cask docker

# 5. Install GitHub CLI
brew install gh
gh auth login

# 6. Install mpak CLI
npm install -g @nimblebrain/mpak

# 7. Verify everything
python3 --version    # 3.13+
uv --version
ruff --version
ty --version
docker --version
gh --version
mpak --version
```

## Linux Setup

```bash
# 1-3: Same as macOS (uv, Python, ruff, ty)
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.13
uv tool install ruff
uv tool install ty

# 4. Docker
# See https://docs.docker.com/engine/install/

# 5. GitHub CLI
sudo apt install gh   # Debian/Ubuntu
gh auth login

# 6. mpak CLI
npm install -g @nimblebrain/mpak
```

## Project Setup

```bash
# Create server repo from template
gh repo create NimbleBrainInc/mcp-server-<name> \
  --template NimbleBrainInc/mcp-server-template --public --clone

cd mcp-server-<name>

# Install dependencies
uv sync --dev

# Verify
uv run ruff check src/ tests/
uv run ty check src/
uv run pytest tests/ -v
```

## Verification Commands (Run Often)

```bash
uv run ruff format src/ tests/       # Format code
uv run ruff check src/ tests/        # Lint
uv run ty check src/                 # Type check
uv run pytest tests/ -v              # Run tests
make check                           # All of the above
```
