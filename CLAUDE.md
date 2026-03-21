# NimbleBrain Contributor Toolkit

Installable skill pack for NimbleBrain open source contributors. Published to mpak as individual skills.

## Structure

```
nimblebrain-contributor/     # Onboarding, coordination, HQ interaction
├── SKILL.md
├── CHANGELOG.md
├── version.txt
└── references/
    ├── DEV_SETUP.md
    ├── ECOSYSTEM.md
    └── RESOURCES.md

build-mcpb/                  # Full MCP server build pipeline
├── SKILL.md
├── CHANGELOG.md
├── version.txt
└── references/
    ├── CONVENTIONS-PY.md
    ├── CONVENTIONS-TS.md
    ├── PATTERNS-PY.md
    ├── PATTERNS-TS.md
    ├── SKILL_FORMAT-PY.md
    ├── SKILL_FORMAT-TS.md
    └── workflow/
        ├── phase-0-bootstrap.md
        ├── phase-1-api-analysis.md
        ├── phase-2-scaffold.md
        ├── phase-3-implement-and-verify.md
        ├── phase-4-validate-bundle.md
        ├── phase-5-embed-skill.md
        └── phase-6-release.md
```

## Versioning

Uses release-please with conventional commits. Each skill versions independently.

**DO NOT manually edit:**
- `version.txt` (release-please only)
- `.release-please-manifest.json` (release-please only)
- `SKILL.md` `metadata.version` (publish workflow syncs it)

**Commit format:** `feat(skill-name): description`

| Commit | Result |
|--------|--------|
| `fix(build-mcpb): typo` | Patch bump |
| `feat(build-mcpb): new feature` | Minor bump |
| `feat(build-mcpb)!: breaking` | Major bump |

## Before Pushing

```bash
./scripts/validate.sh
```

## Publishing Flow

1. Edit skill content, commit with conventional format
2. Push to main
3. release-please creates release PR
4. Merge PR -> automatic release + publish to mpak registry
