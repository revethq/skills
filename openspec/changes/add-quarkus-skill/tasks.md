# Implementation Tasks

## 1. Create Quarkus Skill

- [x] 1.1 Create `revet-skills/skills/quarkus/SKILL.md` with YAML frontmatter and content covering:
  - Tech stack versions (Quarkus 3.31.1, Kotlin 2.3.10, JVM 25, Gradle 9.3.1, ktlint)
  - Multi-module project structure (core/web/persistence)
  - Module-specific configuration (library vs application plugins)
  - Known workarounds (Quarkus bug #36506, Gradle 9 Convention API)
  - Conformance checklist and remediation patterns

## 2. Update Documentation

- [x] 2.1 Update `revet-skills/README.md` to include quarkus skill in Available Skills table
- [x] 2.2 Update installation examples to include quarkus skill

## 3. Validate

- [x] 3.1 Run `openspec validate add-quarkus-skill --strict` and fix any issues
- [x] 3.2 Verify SKILL.md has proper YAML frontmatter (name, description)
- [x] 3.3 Confirm README includes installation instructions for quarkus skill
