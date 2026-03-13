---
name: revet-core
description: Shared domain models for Revet libraries - Metadata, Identifier, SchemaValidation, Organization, Project, Tag, TaggedItem, Urn
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
**Version:** `0.2.1`

### Gradle

```kotlin
// Core domain models (no Quarkus dependency)
implementation("com.revethq:revet-core-core:0.2.1")

// Persistence layer (Panache entities + repositories)
implementation("com.revethq:revet-core-persistence-runtime:0.2.1")

// Web layer (DTOs + mappers)
implementation("com.revethq:revet-core-web:0.2.1")
```

## Domain Models (core module)

### Urn

```kotlin
data class Urn(
    val namespace: String,
    val service: String,
    val tenant: String,
    val resourceType: String,
    val resourceId: String,
) {
    fun matches(pattern: Urn): Boolean
    fun matches(pattern: String): Boolean
    override fun toString(): String  // "urn:{namespace}:{service}:{tenant}:{resourceType}/{resourceId}"

    companion object {
        fun parse(urn: String): Urn?
        fun parseOrThrow(urn: String): Urn
    }
}
```

Uniform Resource Name for globally unique identification of resources. Format: `urn:{namespace}:{service}:{tenant}:{resourceType}/{resourceId}`. Supports wildcard pattern matching: `*` matches a single path segment, `**` matches zero or more segments. Wildcards work in all components.

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

### Tag

```kotlin
data class Tag(
    val id: Int?,
    val name: String,
    val slug: String,
    val organizationId: Long,
) {
    companion object {
        fun create(name: String, organizationId: Long, slug: String? = null): Tag
    }

    fun update(name: String? = null, slug: String? = null): Tag
    fun isNew(): Boolean
}
```

Organization-scoped tag with auto-generated slug (lowercased, non-alphanumeric replaced with hyphens). Factory method `create()` generates slug from name if not provided.

### TaggedItem

```kotlin
data class TaggedItem(
    val id: Long?,
    val tagId: Int,
    val resourceUrn: String,
) {
    companion object {
        fun create(tagId: Int, resourceUrn: String): TaggedItem
    }

    fun isNew(): Boolean
}
```

Generic association between a Tag and any entity identified by a URN. Enables polymorphic tagging — any resource with a URN can be tagged without schema changes.

## Persistence Layer (persistence-runtime module)

### Entities

- `OrganizationEntity` - Panache entity mapped to `revet_organizations` table. Contact info fields are flattened columns.
- `ProjectEntity` - Panache entity mapped to `revet_projects` table. Uses `@ManyToOne` to OrganizationEntity and `@ElementCollection` for `clientIds` (table `project_clients`) and `tags` (table `project_tags`).
- `TagEntity` - Panache entity mapped to `revet_tags` table. Fields: `id` (Int), `name`, `slug`, `organizationId`.
- `TaggedItemEntity` - Panache entity mapped to `revet_tagged_items` table. Uses `@ManyToOne` to TagEntity and stores `resourceUrn` as a string column.

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

```kotlin
interface TagRepository {
    fun findAll(): List<Tag>
    fun findById(id: Int): Tag?
    fun findByName(name: String, organizationId: Long): Tag?
    fun findBySlug(slug: String, organizationId: Long): Tag?
    fun findAllByOrganizationId(organizationId: Long): List<Tag>
    fun save(tag: Tag): Tag
    fun delete(id: Int): Boolean
}

interface TaggedItemRepository {
    fun addTagToResource(tagId: Int, resourceUrn: String)
    fun removeTagFromResource(tagId: Int, resourceUrn: String): Boolean
    fun findTagsByResource(resourceUrn: String): List<Tag>
    fun deleteByTagId(tagId: Int)
}
```

All repositories have `@ApplicationScoped` Panache implementations with `@Transactional` write operations. Organization and Project support soft-delete. Tag uses hard delete. TaggedItem operations are idempotent.

### Mappers

- `OrganizationMapper` - Converts between `OrganizationEntity` and `Organization` domain model. Handles `ContactInfo` composition.
- `ProjectMapper` - Converts between `ProjectEntity` and `Project` domain model. Handles `Timestamps` composition.
- `TagMapper` - Converts between `TagEntity` and `Tag` domain model.
- `TaggedItemMapper` - Converts `TaggedItemEntity` to `TaggedItem` domain model.

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

**Tag:**
- `TagDTO` - Response DTO with id, name, slug, organizationId
- `CreateTagRequest` - Creation request (name required, slug optional)
- `UpdateTagRequest` - Partial update request (name and slug optional)

### Mappers

- `OrganizationDTOMapper` - Maps `Organization` domain to/from DTOs. Includes `toContactInfo()` helpers for request conversion.
- `ProjectDTOMapper` - Maps `Project` domain to `ProjectDTO`.
- `TagDTOMapper` - Maps `Tag` domain to `TagDTO`.

## Characteristics

- **Immutable by Default**: Domain models are data classes; all mutations return new copies
- **Soft Delete**: Both Organization and Project use `isActive` + timestamp pattern
- **Framework Independence**: Core module has zero framework dependencies
- **Layered Architecture**: Domain ↔ Entity ↔ DTO with explicit mappers at each boundary

## Used By

- `revet-iam` - User, Group, Policy entities include `metadata: Metadata`
- `revet-auth` - Application, Client, Scope entities use Metadata
- `revet-documents` - Consumes Organization and Project models
