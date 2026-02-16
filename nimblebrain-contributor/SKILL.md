---
name: nimblebrain-contributor
description: Onboard into NimbleBrain open source and coordinate contributions. Checks your development environment, orients you on what NimbleBrain builds and how we work, helps you pick available work from open issues, file issues, and get started on your first contribution. Use when joining NimbleBrain, looking for work, or interacting with project coordination. Triggers include "I'm a new contributor", "onboard me", "what should I build", "show me open issues", "file an issue".
license: Apache-2.0
compatibility: Requires gh (GitHub CLI) authenticated with GitHub. Python 3.13+, uv, ruff, ty for MCP server work.
allowed-tools: Read Write Bash Glob Grep WebFetch AskUserQuestion
metadata:
  tags:
    - onboarding
    - contributor
    - nimblebrain
    - coordination
  category: development
  triggers:
    - I'm a new contributor
    - onboard me
    - help me get started
    - what should I build
    - check open issues
    - file an issue
    - what's available to work on
    - show me open issues
  keywords:
    - onboarding
    - contributor
    - nimblebrain
    - getting-started
    - issues
  version: "0.1.0"
  surfaces:
    - claude-code
  author:
    name: NimbleBrain
    url: https://nimblebrain.ai
  examples:
    - prompt: I'm a new contributor, help me get started
      context: First day contributing to NimbleBrain
    - prompt: What integrations are available to build?
      context: Looking for work to pick up
    - prompt: File an issue for a Slack MCP server
      context: Proposing a new integration
---

# NimbleBrain Contributor

Onboard into NimbleBrain open source and coordinate your contributions.

## What is NimbleBrain?

NimbleBrain is a business automation platform powered by AI. Users talk to Nira (the AI agent) to automate workflows using tools they already use: Notion, HubSpot, Linear, Stripe, etc.

The open source ecosystem has three layers:
- **MCP Servers** connect to external services and expose tools (MCPB bundles published to mpak)
- **Skills** compose those tools into reusable workflows (published as .skill packages)
- **Platform** (Studio) orchestrates everything for teams

Most open source contribution is building MCP servers + companion skills.

## Process

### If the user is new: Full Onboarding

#### Step 1: Check Environment

Verify these tools are installed. For each missing tool, provide the install command.

```bash
python3 --version          # Need 3.13+
uv --version               # Package manager
ruff --version              # Linter/formatter
ty --version                # Type checker
docker --version            # For testing bundles
gh --version                # GitHub CLI
gh auth status              # Must be authenticated
```

**Install commands for missing tools:**

| Tool | macOS | Linux |
|------|-------|-------|
| Python 3.13 | `uv python install 3.13` | `uv python install 3.13` |
| uv | `curl -LsSf https://astral.sh/uv/install.sh \| sh` | same |
| ruff | `uv tool install ruff` | same |
| ty | `uv tool install ty` | same |
| Docker | `brew install --cask docker` | See docs.docker.com |
| gh | `brew install gh` | `sudo apt install gh` |

Also check for mpak CLI:
```bash
mpak --version
```
If missing: `npm install -g @nimblebrain/mpak`

#### Step 2: Check Required Skills

Verify the build-mcpb skill is installed:
```bash
ls ~/.claude/skills/build-mcpb/SKILL.md 2>/dev/null && echo "installed" || echo "MISSING"
```

If missing: `mpak skill install @nimblebraininc/build-mcpb`

#### Step 3: Orient

Explain what NimbleBrain builds and how contributions work:

```
How NimbleBrain Contributions Work
====================================

1. Pick an integration from open issues (or propose your own)
2. Create a server repo from the template
3. Run /build-mcpb to scaffold, implement, validate, and author skills
4. Submit a PR with server + companion skills
5. CI validates, maintainer reviews, then it ships to mpak

Every deliverable is a server + skills combo:
  - MCP server with 5+ tools
  - 2-3 companion skills that compose those tools into workflows
```

#### Step 4: Help Pick Work

Browse open issues:

```bash
# Check open issues on HQ
gh issue list --repo NimbleBrainInc/hq

# Check for good first issues
gh issue list --repo NimbleBrainInc/hq --label "good first issue"

# Check available integrations
gh issue list --repo NimbleBrainInc/hq --label "integration"
```

Present the options and help the user pick something that matches their interest.

#### Step 5: Get Started

Once they've picked an integration:

1. Create the server repo:
   ```bash
   gh repo create NimbleBrainInc/mcp-<name> \
     --template NimbleBrainInc/mcp-server-template --public --clone
   cd mcp-<name>
   ```

2. Get their API key for the target service

3. Hand off to `/build-mcpb` to start the build pipeline

### If the user wants to browse work

Browse open issues and present a summary of what's available:

```bash
gh issue list --repo NimbleBrainInc/hq --state open
gh issue list --repo NimbleBrainInc/hq --label "integration"
```

### If the user wants to file an issue

Help them create a well-structured issue on HQ:

**New integration proposal:**
```bash
gh issue create --repo NimbleBrainInc/hq \
  --title "New MCP Server: <service name>" \
  --body "## Service
- **API docs:** <url>
- **Auth:** <method>
- **Free tier:** <yes/no>

## Suggested Tools
- \`tool_1\` - description
- \`tool_2\` - description

## Suggested Skills
1. \`skill-name\` - description

## Notes
<any API quirks, complexity notes, etc.>" \
  --label "integration"
```

**Bug report:**
```bash
gh issue create --repo NimbleBrainInc/hq \
  --title "<concise description>" \
  --body "## What happened
<description>

## Expected behavior
<description>

## Steps to reproduce
1. ..." \
  --label "bug"
```

### If the user wants to check status

```bash
gh issue list --repo NimbleBrainInc/hq --state all --limit 20
gh issue list --repo NimbleBrainInc/hq --assignee @me
```

## Key Links

- **HQ repo:** github.com/NimbleBrainInc/hq (issues, roadmap)
- **mpak registry:** mpak.dev (published servers and skills)
- **Docs:** docs.nimblebrain.ai
- **MCP spec:** modelcontextprotocol.io

## References

See `references/` in this skill for:
- `DEV_SETUP.md` - Detailed environment setup instructions
- `PIPELINE.md` - Full build-validate-author-publish pipeline reference
