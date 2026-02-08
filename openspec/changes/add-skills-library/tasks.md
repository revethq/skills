## 1. Repository Setup

- [x] 1.1 Create `revet-skills` repository
- [x] 1.2 Add README.md with usage instructions
- [x] 1.3 Set up directory structure (`skills/iam/`, `skills/core/`, `skills/auth/`)

## 2. IAM Skill

- [x] 2.1 Create `skills/iam/SKILL.md` with frontmatter, module structure, and dependency coordinates
- [x] 2.2 Create `skills/iam/permissions.md` with URN format spec, Policy/Statement/Condition class signatures, evaluation semantics
- [x] 2.3 Create `skills/iam/users.md` with User/Profile/Group/GroupMember data class definitions and service interfaces
- [x] 2.4 Create `skills/iam/scim.md` with SCIM DTO definitions, endpoint contracts, filter grammar
- [x] 2.5 Document extension points and customization hooks
- [x] 2.6 Document correct usage patterns for common integration scenarios
- [x] 2.7 Document error conditions and edge cases

## 3. Core Skill

- [x] 3.1 Create `skills/core/SKILL.md` with frontmatter and overview
- [x] 3.2 Document shared domain models and their usage

## 4. Auth Skill

- [x] 4.1 Create `skills/auth/SKILL.md` with frontmatter and overview
- [x] 4.2 Document OAuth 2.1 / OIDC integration patterns (when library is production-ready)

## 5. Validation

- [ ] 5.1 Test installation via `npx skills add github:revethq/revet-skills --skill iam`
- [ ] 5.2 Verify skills load correctly in Claude Code
- [ ] 5.3 Verify skills load correctly in at least one other agent (Cursor or Codex)
- [ ] 5.4 Test discovery via `npx skills find revet`

## 6. Repository Documentation

- [x] 6.1 Document skills installation in revet-skills README
- [x] 6.2 Document versioning/tagging strategy
- [x] 6.3 Add contribution guidelines for maintaining skill accuracy
