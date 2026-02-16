# Skill Format Quick Reference

## Directory Structure

```
my-skill/
├── SKILL.md              # Required: frontmatter + instructions
├── scripts/              # Optional: executable code
├── references/           # Optional: supplementary docs
└── assets/               # Optional: static resources
```

## Required Frontmatter

| Field | Constraints |
|-------|-------------|
| `name` | 1-64 chars, lowercase alphanumeric + hyphens, must match directory name |
| `description` | 1-1024 chars, third person, what it does AND when to use it |

## Optional Frontmatter

| Field | Description |
|-------|-------------|
| `license` | License identifier (e.g., MIT, Apache-2.0) |
| `compatibility` | Environment requirements (max 500 chars) |
| `allowed-tools` | Space-delimited tool list (e.g., "Read Write Bash") |

## Discovery Metadata (via `metadata:`)

| Field | Type | Constraints |
|-------|------|-------------|
| `tags` | string[] | Max 10 tags, each max 32 chars |
| `category` | enum | development, writing, research, consulting, data, design, operations, security, other |
| `triggers` | string[] | Max 20, each max 128 chars |
| `keywords` | string[] | Max 30, each max 32 chars |
| `version` | string | Semver (required for registry publishing) |
| `surfaces` | enum[] | claude-code, claude-api, claude-ai |
| `author` | object | {name (required), url, email} |
| `examples` | object[] | Max 5, each has prompt (required) and context (optional) |

## Naming Convention

Scoped names for registry: `@scope/skill-name`
- Scope matches GitHub org (e.g., `@nimblebraininc/`)
- Lowercase alphanumeric + hyphens only

## Validation and Publishing

```bash
mpak skill validate ./my-skill     # Validate
mpak skill pack ./my-skill         # Create .skill bundle
# Publishing happens via GitHub Actions (skill-pack action)
```
