# Revet Skills

AI-optimized context for Revet library integrations. These skills provide structured context for coding agents (Claude Code, Cursor, Codex, Copilot, etc.) when working with Revet libraries.

## Installation

Using the Vercel Skills CLI:

```bash
# Install IAM skill
npx skills add github:revethq/revet-skills --skill iam

# Install Core skill
npx skills add github:revethq/revet-skills --skill core

# Install Auth skill
npx skills add github:revethq/revet-skills --skill auth

# Install Quarkus skill
npx skills add github:revethq/revet-skills --skill quarkus
```

Pin to a specific version:

```bash
npx skills add github:revethq/revet-skills@iam-0.1.13 --skill iam
```

## Available Skills

| Skill | Library | Description |
|-------|---------|-------------|
| `iam` | revet-iam | Identity and access management: permissions, users, groups, SCIM |
| `core` | revet-core | Shared domain models: Metadata, Identifier, SchemaValidation |
| `auth` | revet-auth | OAuth 2.1 / OIDC authorization server (in development) |
| `quarkus` | - | Project structure and configuration for Quarkus/Kotlin multi-module projects |

## Discovery

```bash
npx skills find revet
```

## Versioning

Skills are tagged to match library releases:
- IAM 0.1.13 → `iam-0.1.13`
- Core 0.1.0 → `core-0.1.0`

## Structure

```
revet-skills/
├── skills/
│   ├── iam/
│   │   ├── SKILL.md          # Entry point
│   │   ├── permissions.md    # URN/Policy system
│   │   ├── users.md          # User/Group management
│   │   └── scim.md           # SCIM integration
│   ├── core/
│   │   └── SKILL.md          # Shared domain models
│   ├── auth/
│   │   └── SKILL.md          # OAuth 2.1/OIDC (preview)
│   └── quarkus/
│       └── SKILL.md          # Quarkus/Kotlin project standards
└── README.md
```

## Contributing

When updating skills to match new library versions:

1. Update skill content to reflect API changes
2. Tag the commit: `git tag iam-X.Y.Z`
3. Push tags: `git push --tags`

Ensure skill content stays synchronized with the corresponding library version. Skills should document public API contracts, not implementation details.

## License

Apache License 2.0
