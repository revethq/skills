## ADDED Requirements

### Requirement: Quarkus Skill Content

The Quarkus skill SHALL provide AI agents with context for creating and auditing Revet Quarkus/Kotlin projects.

#### Scenario: Tech stack version context

- **WHEN** an agent creates or audits a Quarkus project
- **THEN** the skill provides current tech stack versions (Quarkus 3.31.1, Kotlin 2.3.10, JVM 25, Gradle 9.3.1)
- **AND** the skill provides ktlint configuration requirements

#### Scenario: Multi-module project structure context

- **WHEN** an agent scaffolds a new multi-module project
- **THEN** the skill provides the standard module layout (core/web/persistence)
- **AND** the skill provides module dependency relationships
- **AND** the skill provides naming conventions for modules

#### Scenario: Module configuration context

- **WHEN** an agent configures Gradle build files
- **THEN** the skill provides library plugin configuration for core modules
- **AND** the skill provides application plugin configuration for web modules
- **AND** the skill provides persistence-specific plugin requirements

#### Scenario: Known workarounds context

- **WHEN** an agent encounters known framework issues
- **THEN** the skill documents Quarkus bug #36506 workaround (dev mode classloading)
- **AND** the skill documents Gradle 9 Convention API migration patterns

#### Scenario: Conformance audit context

- **WHEN** an agent audits an existing project for conformance
- **THEN** the skill provides a checklist of required configurations
- **AND** the skill provides remediation patterns for common violations
- **AND** the skill identifies deviation from standard module structure
