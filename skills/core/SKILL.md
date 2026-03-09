---
name: revet-core
description: Shared domain models for Revet libraries - Metadata, Identifier, SchemaValidation, Organization, Project
---

# Revet Core Library

Shared domain models and supporting layers used across Revet libraries. Multi-module project providing core domain models, persistence (Panache), and web (DTOs/mappers).

## Modules

| Module | Artifact ID | Purpose |
|--------|-------------|---------|
| `core` | `revet-core-core` | Domain models, no framework dependencies |
| `persistence-runtime` | `revet-core-persistence-runtime` | Panache entities, repositories, entity-domain mappers |
| `web` | `revet-core-web` | DTOs, DTO-domain mappers |

## Dependency Coordinates

**Group ID:** `com.revethq`
**Version:** `0.2.0`

### Gradle

```kotlin
// Core domain models (no Quarkus dependency)
implementation("com.revethq:revet-core-core:0.2.0")

// Persistence layer (Panache entities + repositories)
implementation("com.revethq:revet-core-persistence-runtime:0.2.0")

// Web layer (DTOs + mappers)
implementation("com.revethq:revet-core-web:0.2.0")
```

## Domain Models (core module)

### Identifier

```kotlin
data class Identifier(
    val system: String? = null,
    val value: String? = null
)
```

Represents an external identifier with a system namespace and value.

### SchemaValidation

```kotlin
data class SchemaValidation(
    val schemaId: UUID? = null,
    val isValid: Boolean = false,
    val validatedOn: OffsetDateTime? = null
)
```

Records schema validation state for a resource.

### Metadata

```kotlin
data class Metadata(
    val identifiers: List<Identifier> = emptyList(),
    val schemaValidations: List<SchemaValidation> = emptyList(),
    val properties: Map<String, Any> = emptyMap()
)
```

Aggregate metadata container combining identifiers, schema validation history, and extensible key-value properties.

### Organization

```kotlin
data class Organization(
    val id: Long?,
    val uuid: UUID,
    val name: String,
    val description: String?,
    val contactInfo: ContactInfo,
    val locale: String?,
    val timezone: String,
    val bucketId: Long?,
    val isActive: Boolean,
    val removedAt: LocalDateTime?,
) {
    data class ContactInfo(
        val address: String?, val city: String?, val state: String?,
        val zipCode: String?, val country: String?, val phone: String?,
        val fax: String?, val website: String?,
    )

    companion object {
        fun create(name: String, ...): Organization
    }

    fun update(name: String? = null, ...): Organization
    fun deactivate(): Organization
    fun isNew(): Boolean
}
```

Business entity with contact information, locale/timezone, and soft-delete lifecycle. Factory method `create()` generates a UUID and sets defaults. `update()` returns an immutable copy with changed fields. `deactivate()` sets `isActive = false` and stamps `removedAt`.

### Project

```kotlin
data class Project(
    val id: Long?,
    val uuid: UUID,
    val name: String,
    val description: String?,
    val organizationId: Long,
    val clientIds: Set<UUID>,
    val tags: Set<String>,
    val isActive: Boolean,
    val timestamps: Timestamps,
) {
    data class Timestamps(
        val createdAt: LocalDateTime,
        val modifiedAt: LocalDateTime,
        val removedAt: LocalDate?,
    )

    companion object {
        fun create(name: String, organizationId: Long, ...): Project
    }

    fun update(name: String? = null, ...): Project
    fun deactivate(): Project
    fun addClient(clientId: UUID): Project
    fun removeClient(clientId: UUID): Project
    fun addTag(tag: String): Project
    fun removeTag(tag: String): Project
    fun isNew(): Boolean
}
```

Scoped work context belonging to an Organization. Manages client UUIDs and tags as immutable sets. All mutating methods return a new copy with `modifiedAt` advanced.

## Persistence Layer (persistence-runtime module)

### Entities

- `OrganizationEntity` - Panache entity mapped to `revet_organizations` table. Contact info fields are flattened columns.
- `ProjectEntity` - Panache entity mapped to `revet_projects` table. Uses `@ManyToOne` to OrganizationEntity and `@ElementCollection` for `clientIds` (table `project_clients`) and `tags` (table `project_tags`).

### Repositories

```kotlin
interface OrganizationRepository {
    fun findAll(includeInactive: Boolean = false): List<Organization>
    fun findById(id: Long): Organization?
    fun findByUuid(uuid: UUID): Organization?
    fun save(organization: Organization): Organization
    fun delete(id: Long): Boolean
}

interface ProjectRepository {
    fun findAll(includeInactive: Boolean = false): List<Project>
    fun findById(id: Long): Project?
    fun findByUuid(uuid: UUID): Project?
    fun findByOrganizationId(organizationId: Long, includeInactive: Boolean = false): List<Project>
    fun save(project: Project): Project
    fun delete(id: Long): Boolean
}
```

Both repositories have `@ApplicationScoped` Panache implementations with `@Transactional` write operations and soft-delete support.

### Mappers

- `OrganizationMapper` - Converts between `OrganizationEntity` and `Organization` domain model. Handles `ContactInfo` composition.
- `ProjectMapper` - Converts between `ProjectEntity` and `Project` domain model. Handles `Timestamps` composition.

## Web Layer (web module)

### DTOs

**Organization:**
- `OrganizationDTO` - Response DTO with contact info flattened to top-level fields
- `CreateOrganizationRequest` - Creation request
- `UpdateOrganizationRequest` - Partial update request (all fields optional)

**Project:**
- `ProjectDTO` - Response DTO with timestamps flattened to `createdAt`/`modifiedAt`
- `CreateProjectRequest` / `UpdateProjectRequest`
- `AddClientRequest` / `RemoveClientRequest`
- `AddTagRequest` / `RemoveTagRequest`

### Mappers

- `OrganizationDTOMapper` - Maps `Organization` domain to/from DTOs. Includes `toContactInfo()` helpers for request conversion.
- `ProjectDTOMapper` - Maps `Project` domain to `ProjectDTO`.

## Characteristics

- **Immutable by Default**: Domain models are data classes; all mutations return new copies
- **Soft Delete**: Both Organization and Project use `isActive` + timestamp pattern
- **Framework Independence**: Core module has zero framework dependencies
- **Layered Architecture**: Domain ↔ Entity ↔ DTO with explicit mappers at each boundary

## Used By

- `revet-iam` - User, Group, Policy entities include `metadata: Metadata`
- `revet-auth` - Application, Client, Scope entities use Metadata
- `revet-documents` - Consumes Organization and Project models
