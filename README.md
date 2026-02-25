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

## Versioning

Each skill versions independently via [release-please](https://github.com/googleapis/release-please) with [conventional commits](https://www.conventionalcommits.org/).

```bash
# Edit a skill, commit with scope
git commit -m "feat(build-mcpb): add GraphQL client support"

# release-please creates a PR, merge it to publish
```

## Links

- [NimbleBrain HQ](https://github.com/NimbleBrainInc/.github) - Issues, coordination
- [MCP Server Template (Python)](https://github.com/NimbleBrainInc/mcp-server-template-python)
- [MCP Server Template (TypeScript)](https://github.com/NimbleBrainInc/mcp-server-template-typescript)
- [mpak Registry](https://mpak.dev)
- [Agent Skills Spec](https://agentskills.io/specification)

## License

Apache 2.0
