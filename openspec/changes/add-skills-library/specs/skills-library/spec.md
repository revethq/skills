## ADDED Requirements

### Requirement: Skills Repository Structure

The skills repository SHALL follow Vercel Skills ecosystem conventions with a `skills/` directory containing one subdirectory per published library.

#### Scenario: Standard directory layout
- **WHEN** examining the repository structure
- **THEN** each library skill is in `skills/<library-name>/`
- **AND** each skill directory contains a `SKILL.md` entry point

#### Scenario: Skill naming convention
- **WHEN** a skill is created for a library
- **THEN** the skill name uses format `revet-<library>` (e.g., `revet-iam`)
- **AND** the directory name matches the library name (e.g., `iam/`)

### Requirement: SKILL.md Entry Point

Each skill SHALL have a `SKILL.md` file with YAML frontmatter containing `name` and `description` fields.

#### Scenario: Valid frontmatter
- **WHEN** reading a SKILL.md file
- **THEN** the file starts with YAML frontmatter delimited by `---`
- **AND** frontmatter contains `name` field matching `revet-<library>`
- **AND** frontmatter contains `description` field summarizing the library's purpose

#### Scenario: Content structure
- **WHEN** reading SKILL.md content after frontmatter
- **THEN** the content includes installation instructions
- **AND** the content includes core concepts overview
- **AND** the content references supporting files for detailed topics

### Requirement: Supporting Documentation Files

Skills SHALL support additional markdown files for detailed topics that would exceed 500 lines in SKILL.md.

#### Scenario: Topic separation
- **WHEN** a library has multiple distinct integration areas
- **THEN** each area is documented in a separate file (e.g., `permissions.md`, `users.md`)
- **AND** SKILL.md references these files with brief descriptions

#### Scenario: File size guideline
- **WHEN** writing skill documentation
- **THEN** individual files SHOULD remain under 500 lines
- **AND** content is split into focused topics when approaching this limit

### Requirement: Cross-Agent Compatibility

Skills SHALL be installable via Vercel Skills CLI (`npx skills`) and compatible with multiple AI coding agents.

#### Scenario: Installation via npx
- **WHEN** a user runs `npx skills add github:revethq/revet-skills --skill iam`
- **THEN** the IAM skill is installed to the appropriate agent-specific location

#### Scenario: Agent discovery
- **WHEN** a user runs `npx skills find revet`
- **THEN** available Revet skills are listed in search results

### Requirement: Library Version Alignment

Skills repository SHALL support version tagging aligned with library releases.

#### Scenario: Version-specific installation
- **WHEN** a user needs skills for a specific library version
- **THEN** they can install using `npx skills add github:revethq/revet-skills@iam-X.Y.Z --skill iam`
- **AND** the installed skill content matches that library version

### Requirement: IAM Skill Content

The IAM skill SHALL provide AI agents with context for generating IAM integration code.

#### Scenario: Permission system context
- **WHEN** an agent generates code using IAM permissions
- **THEN** the skill provides URN format specification and valid patterns
- **AND** the skill provides Policy and Statement class signatures
- **AND** the skill provides Condition types and their evaluation semantics
- **AND** the skill provides AuthorizationRequest/Result contracts

#### Scenario: User management context
- **WHEN** an agent generates code using IAM user management
- **THEN** the skill provides User and Profile data class definitions
- **AND** the skill provides Group and GroupMember data class definitions
- **AND** the skill provides service interface method signatures

#### Scenario: SCIM integration context
- **WHEN** an agent generates code integrating with SCIM
- **THEN** the skill provides SCIM DTO class definitions
- **AND** the skill provides resource endpoint paths and methods
- **AND** the skill provides filter syntax grammar

### Requirement: Agent-Optimized Content Format

Skill content SHALL be structured for AI agent consumption.

#### Scenario: Explicit type signatures
- **WHEN** documenting a public interface
- **THEN** the skill includes complete method signatures with parameter and return types
- **AND** the skill uses code-like definitions over prose descriptions

#### Scenario: Constraint specification
- **WHEN** a library enforces invariants or constraints
- **THEN** the skill states valid input ranges and expected behaviors
- **AND** the skill states exception conditions

#### Scenario: Usage patterns
- **WHEN** documenting integration scenarios
- **THEN** the skill provides correct usage patterns
- **AND** the skill shows expected method call sequences

#### Scenario: Error conditions
- **WHEN** a library has known failure modes or edge cases
- **THEN** the skill documents exception types and when they occur
- **AND** the skill documents edge case behaviors
