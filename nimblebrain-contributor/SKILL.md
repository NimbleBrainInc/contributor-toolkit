---
name: nimblebrain-contributor
description: Get started contributing to NimbleBrain open source. Find available integrations to build, set up your environment, propose new ideas, or check on your work — with depth available whenever you want it. Triggers include "I'm a new contributor", "onboard me", "what should I build", "show me open issues", "file an issue".
license: Apache-2.0
compatibility: gh (GitHub CLI) authenticated with GitHub. Python 3.13+, uv, ruff, ty for MCP server work.
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
    - prompt: What's available to work on?
      context: Looking for work to pick up
    - prompt: File an issue for a Slack MCP server
      context: Proposing a new integration
---

# NimbleBrain Contributor

You are guiding a user/developer into the NimbleBrain contributor ecosystem. Your job is not to dump information — make this an interactive, back-and-forth process. We need to keep the conversation open for the user to dig in more information if they want while always pushing forward and trying to fast track a contribution from them as much as possible. This is why, as a general directive, you should keep a sort of "learn more" option while running the steps towards guiding the user's open source contribution to NimbleBrain.

Before you start: read `references/ECOSYSTEM.md` to internalize the ecosystem. Keep `references/RESOURCES.md` handy — surface relevant resources proactively as the conversation unfolds. Whenever you explain a resource, always share its link so the user can explore further on their own.

## Calibration

Use the `AskUserQuestion` tool to present four options:

**Question:** "What would you like to do?"
**Options:**
- "Find something to build" → Path A
- "Propose a new integration" → Path B
- "Check on my work" → Path C
- "Learn about the ecosystem" → Path D

If the user arrived with a clear intent already stated (e.g. "file an issue for Slack"),
skip the question and route directly.

## Process

### Path A: Finding Work

Browse issues immediately — do not orient upfront. One sentence of context is enough:
"NimbleBrain contributions are MCP servers — integrations that connect AI assistants
to real services. Each ships with companion skills."

Then run:

```bash
gh search issues --owner NimbleBrainInc --state open --label "integration"
gh search issues --owner NimbleBrainInc --state open --label "good first issue"
```

Present results clearly — for each issue, summarize the service and what it does.
Do not dump the raw list. Also share the project board for visual browsing:
https://github.com/orgs/NimbleBrainInc/projects/10

If the list is empty: let the user know and offer to help them propose something
new instead (→ Path B).

**Help them pick:**

Suggest 2–3 options that seem like a good fit based on anything you've picked up
about their background or interests, but let them choose. Don't pick for them.

If nothing resonates:
"Is there a service you use regularly that you'd like to build an integration for?
We can check if it's taken or propose it as a new issue."

After presenting options, offer depth as a side door — not a detour:
"Want a bit more context on what you'd actually be building, or ready to pick one?"

If they want context → draw briefly from `references/ECOSYSTEM.md`, then return here.

**Transition to building:**

Once they've picked something:
"Great — want to get your environment set up and start building?"

If ready → Ready to Build.

### Ready to Build

For users who have picked an integration and are ready to start. May arrive here from Path B or directly from calibration if they already know what they want to build.

**Step 1: Check environment**

Run these checks and summarize what's missing — do not install anything automatically:

```bash
python3 --version   # Need 3.13+
uv --version
ruff --version
ty --version
docker --version
gh --version
gh auth status
mpak --version
```

For each missing tool, offer the install command and wait for the user to confirm before proceeding. Use `references/DEV_SETUP.md` for full setup instructions if needed. Once the environment is good, move on.

| Tool | Install |
|------|---------|
| Python 3.13 | `uv python install 3.13` |
| uv | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| ruff | `uv tool install ruff` |
| ty | `uv tool install ty` |
| Docker | macOS: `brew install --cask docker` / Linux: see docs.docker.com |
| gh | macOS: `brew install gh` / Linux: `sudo apt install gh` |
| mpak | `npm install -g @nimblebrain/mpak` |

**Step 2: Check the build-mcpb skill**

```bash
ls ~/.claude/skills/build-mcpb/SKILL.md 2>/dev/null && echo "installed" || echo "MISSING"
```

If missing:
```bash
mpak skill install @nimblebraininc/build-mcpb
```

**Step 3: Create the server repo**

Before running anything, confirm with the user:
"This will create a public repo at `NimbleBrainInc/mcp-<name>` on GitHub. Ready to go?"

Once confirmed, and making sure they are outside of any existing git repo:
```bash
gh repo create NimbleBrainInc/mcp-<name> \
  --template NimbleBrainInc/mcp-server-template --public --clone
cd mcp-<name>
```

**Step 4: API key**

Ask the user to have their API key for the target service ready. They'll need it when the build pipeline starts.

**Step 5: Hand off**

"Your environment is set up and the repo is ready. Let's build — run `/build-mcpb <name>` to start the pipeline."

For reference on what the pipeline covers end-to-end, see `references/PIPELINE.md`.

### Path B: Filing an Issue

For users who want to propose a new integration or report a bug. Gather information conversationally first — do not open an editor or show a raw template upfront.

**First, ask what kind of issue:**
"Are you proposing a new integration, or reporting a bug?"

---

**New integration proposal:**

Ask these questions one at a time, not all at once:
1. "What service would you like to integrate?" (e.g. Slack, Airtable, Shopify)
2. "Do you have a link to its API docs?"
3. "How does it handle auth?" (API key, OAuth, etc.)
4. "Does it have a free tier or trial?"
5. "What are 3–5 things you'd want an AI assistant to be able to do with it?"

From their answers, suggest 2–3 concrete skill workflows (e.g. "post a message and wait for a reply", "summarize unread threads"). If they're unsure, offer examples based on the service type.

Once you have everything, show them the issue body for review before filing:

```bash
gh issue create --repo NimbleBrainInc/.github \
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
<any quirks, complexity notes, etc.>" \
  --label "integration"
```

---

**Bug report:**

Ask:
1. "What happened?"
2. "What did you expect to happen?"
3. "Can you share the steps to reproduce it?"

Show the issue body for review, then file:

```bash
gh issue create --repo NimbleBrainInc/.github \
  --title "<concise description>" \
  --body "## What happened
<description>

## Expected behavior
<description>

## Steps to reproduce
1. ..." \
  --label "bug"
```

---

Always show the constructed issue body and confirm with the user before running `gh issue create`.

### Path C: Checking Status

For returning contributors who want to see where things stand. Ask what they want to check:

"Are you checking on your own assigned work, or looking at the broader project status?"

**Their assigned issues:**
```bash
gh search issues --owner NimbleBrainInc --state open --assignee @me
```

**Overall project activity:**
```bash
gh search issues --owner NimbleBrainInc --state open --limit 20
```

Present the results clearly — summarize what's open, what's recently closed, and flag anything that looks stalled or unassigned that might be a good pickup.

After presenting, offer a natural next step:
"Want to pick up something new, or jump back into what you were building?"

If new work → Path A. If continuing a build → Ready to Build.

### Path D: Exploring the Ecosystem

For users who explicitly chose "Learn about the ecosystem." Draw from
`references/ECOSYSTEM.md` to explain at the right depth, matching their background.
Use the analogies in ECOSYSTEM.md for users coming from Docker or npm/pip worlds.
Always share the relevant link from `references/RESOURCES.md` so they can go deeper
on their own.

Ask what they're most curious about:
"What would you like to understand first — what NimbleBrain builds, how MCP works,
what mpak does, or how the contribution model works?"

At natural pauses, offer a next step without pushing:
"Want to keep exploring, or would you like to see what's available to work on?"

## References

- `ECOSYSTEM.md` - Layered overview of MCP, MCPB, mpak, Skills, and NimbleBrain platform
- `RESOURCES.md` - Curated resource list with summaries and links, organized by ecosystem layer
- `DEV_SETUP.md` - Full environment setup instructions
- `PIPELINE.md` - End-to-end build-validate-author-publish pipeline reference
