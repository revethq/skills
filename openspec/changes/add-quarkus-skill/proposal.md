# Change: Add revet-quarkus Skill

## Why

Revet's Quarkus-based projects follow consistent patterns (core/web/persistence modules). AI agents need structured context to:
1. Create new projects/modules following these patterns
2. Audit existing projects for conformance with standards
3. Apply common workarounds (Quarkus dev mode bugs, Gradle 9 compatibility)

## What Changes

- New skill: `revet-quarkus` with project structure, module configuration, and conformance guidance
- ADDED requirement in skills-library spec for Quarkus skill content

## Impact

- Affected specs: skills-library
- Affected code: revet-skills/skills/quarkus/SKILL.md, revet-skills/README.md
