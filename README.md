# NimbleBrain Contributor Toolkit

Claude Code skills for contributing to NimbleBrain open source. Install once, use anywhere.

## Install

```bash
npm install -g @nimblebrain/mpak
mpak skill install @nimblebraininc/nimblebrain-contributor
mpak skill install @nimblebraininc/build-mcpb
```

## Get Started

Open Claude Code in any directory:

```bash
claude
> I'm a new contributor, help me get started
```

## Skills

| Skill | Purpose |
|-------|---------|
| `nimblebrain-contributor` | Onboard, browse issues, file issues, pick work, check status |
| `build-mcpb` | Full MCP server lifecycle: scaffold, implement, validate, author skills, PR |

## The Flow

```
1. Install the toolkit                    (30 sec)
2. "I'm a new contributor"               (env check, orientation)
3. Pick an integration from open issues    (browse issues)
4. /build-mcpb                            (scaffold → implement → validate → skills → PR)
```

## Developing Locally

To iterate on skills without publishing to mpak on every change, symlink the repo
folders directly into Claude Code's skills directory.

**1. Backup existing installs**

Move (not copy) existing installs out of `~/.claude/skills/` first. Use a name that
does not share a prefix with the real skill name — Claude Code will pick up any folder
in that directory as a skill, including backups.

```bash
mkdir -p ~/claude-skills-backup
mv ~/.claude/skills/nimblebrain-contributor ~/claude-skills-backup/
mv ~/.claude/skills/build-mcpb ~/claude-skills-backup/
```

**2. Symlink the repo folders**

```bash
ln -s /path/to/contributor-toolkit/nimblebrain-contributor ~/.claude/skills/nimblebrain-contributor
ln -s /path/to/contributor-toolkit/build-mcpb ~/.claude/skills/build-mcpb
```

Any edit you make in the repo is immediately live — no reinstall needed. Restart
Claude Code to pick up changes.

**3. Git branching**

Since the skills are symlinked to the repo, switching branches instantly changes
what the live skill sees. Useful for testing experimental changes against main.

## Versioning

Each skill versions independently via [release-please](https://github.com/googleapis/release-please) with [conventional commits](https://www.conventionalcommits.org/).

```bash
# Edit a skill, commit with scope
git commit -m "feat(build-mcpb): add GraphQL client support"

# release-please creates a PR, merge it to publish
```

## Links

- [NimbleBrain HQ](https://github.com/NimbleBrainInc/hq) - Issues, coordination
- [MCP Server Template](https://github.com/NimbleBrainInc/mcp-server-template)
- [mpak Registry](https://mpak.dev)
- [Agent Skills Spec](https://agentskills.io/specification)

## License

Apache 2.0
