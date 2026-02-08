# Change: Add Skills Library for Revet Libraries

## Why

Revet publishes libraries to Maven Central (IAM, Core, Auth) that are consumed by downstream applications. AI coding agents assisting with these integrations benefit from structured context about library APIs, conventions, and usage patterns. A skills library provides agent-optimized context that enables accurate code generation for Revet library integrations.

## What Changes

- **New repository**: Create `revet-skills` repository following Vercel Skills ecosystem conventions
- **Skill structure**: One skill per published library (iam, core, auth) with SKILL.md entry points
- **Content**: Agent-optimized context covering public API contracts, type signatures, extension points, and constraints
- **Distribution**: Compatible with `npx skills add` CLI for cross-agent support (Claude Code, Cursor, Codex, etc.)

## Impact

- Affected specs: New `skills-library` capability
- Affected code: None (new repository, does not modify existing code)
- Affected projects: All published Revet libraries will have corresponding skills

## Distribution Approach

Uses Vercel Skills ecosystem (`npx skills`) for:
- Cross-agent compatibility (35+ coding agents)
- Standard installation: `npx skills add github:revethq/revet-skills --skill iam`
- Discovery via `npx skills find`
- No custom tooling required

## Initial Scope

Start with IAM library skill covering:
- Maven/Gradle dependency coordinates and module structure
- Public interfaces and their method signatures
- Domain model contracts (URNs, Policies, Users, Groups)
- Extension points and customization hooks
- Correct usage patterns
- Error conditions and edge cases
