## Context

Revet libraries are published to Maven Central for consumption by downstream Kotlin/Quarkus applications. AI coding agents assisting with these integrations benefit from structured context about library APIs and conventions. Human developers have documentation; AI agents need structured context optimized for code generation.

**Stakeholders**: AI coding assistants (Claude Code, Cursor, Codex, Copilot, etc.)

**Constraints**:
- Must work across multiple AI coding agents (not just Claude Code)
- Content must be structured for AI consumption (explicit contracts, not prose)
- Must be maintainable alongside library releases
- Must follow established community conventions

## Goals / Non-Goals

**Goals**:
- Provide AI agents with accurate library context for code generation
- Document public API contracts with exact signatures and types
- Specify constraints, invariants, and extension points
- Cross-agent compatibility via standard tooling

**Non-Goals**:
- Human-readable tutorials or guides (developers have docs)
- Explaining "why" or teaching concepts (agents need "what" and "how")
- Code generation templates (agents generate code; skills provide context)
- Runtime integration (development-time only)

## Decisions

### 1. Repository Structure

```
revet-skills/
├── skills/
│   ├── iam/
│   │   ├── SKILL.md              # Entry point with frontmatter
│   │   ├── permissions.md        # URN/Policy system details
│   │   ├── users.md              # User/Group management
│   │   └── scim.md               # SCIM integration
│   ├── core/
│   │   └── SKILL.md
│   └── auth/
│       └── SKILL.md
└── README.md
```

**Rationale**: Follows Vercel Skills convention. Each library gets its own skill directory with supporting files for detailed topics.

### 2. SKILL.md Format

```yaml
---
name: revet-iam
description: Integration guide for Revet IAM library - identity, access management, and permission evaluation
---

# Revet IAM Library
...
```

**Rationale**: Standard frontmatter enables discovery and cross-agent compatibility.

### 3. Content Organization (Agent-Optimized)

Each skill covers:
1. **Dependency Coordinates** - Exact Maven/Gradle artifacts and current version
2. **Public API Contracts** - Interface signatures, method parameters, return types
3. **Domain Model Definitions** - Data classes, enums, their fields and constraints
4. **Extension Points** - Where and how consumers customize behavior
5. **Invariants and Constraints** - Valid inputs, expected behaviors
6. **Usage Patterns** - Correct approaches for common integration scenarios
7. **Error Conditions** - Exception types, edge cases, failure modes

**Rationale**: Agents benefit from precise, unambiguous information. Explicit contracts over prose descriptions.

### 4. Distribution via Vercel Skills Ecosystem

**Chosen**: `npx skills add github:revethq/revet-skills --skill iam`

**Alternatives considered**:
- Personal clone to `~/.claude/skills/`: Simple but no cross-agent support, manual updates
- Plugin marketplace: More infrastructure overhead, approval gates
- Git submodule: Submodules are complex, project-scoped only

**Rationale**: Vercel Skills provides cross-agent support (35+ agents), standard tooling, and discovery without custom infrastructure.

### 5. Versioning Strategy

Skills repository tagged alongside library releases:
- IAM 0.1.13 → revet-skills tag `iam-0.1.13`
- Users can pin: `npx skills add github:revethq/revet-skills@iam-0.1.13 --skill iam`

**Rationale**: Ensures skill content matches library version being consumed.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Skills drift from library reality | Include in library release checklist; CI validation |
| Vercel Skills ecosystem changes | Standard markdown format; easy to migrate |
| Content becomes stale | Link to authoritative specs; keep guidance high-level |
| Overwhelming detail in SKILL.md | Split into focused supporting files (<500 lines each) |

## Migration Plan

No migration—new repository. Libraries continue publishing to Maven Central unchanged.

## Open Questions

1. Should skills include full interface definitions (copy from source) or summarized signatures?
2. How much internal implementation detail helps agents vs. causes confusion?
3. Should skills be auto-generated from source code annotations or hand-curated?
