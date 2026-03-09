# Service Accounts Module

Package: `com.revethq.iam.serviceaccount`

Non-person entities representing background services, CI/CD tools, and automated systems. Authenticate via OAuth 2.1 Client Credentials flow (handled by a separate auth server). Supports policy attachment and profile management via the same mechanisms as users.

## Domain Model

### ServiceAccount

```kotlin
package com.revethq.iam.serviceaccount.domain

data class ServiceAccount(
    var id: UUID,
    var name: String,
    var description: String? = null,
    var tenantId: String? = null,
    var metadata: Metadata = Metadata(),
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
) {
    fun toUrn(): String =
        "urn:revet:iam:${tenantId.orEmpty()}:service-account/$id"
}
```

### Key Properties

- `name` is display-only — not unique, never used for lookup
- `tenantId` is optional — scopes the service account for multi-tenant deployments
- `toUrn()` constructs the principal URN for policy attachment
- `metadata` is a JSONB property bag (`com.revethq.core.Metadata`)

### URN Format

```
urn:revet:iam:{tenantId}:service-account/{id}
```

Examples:
- `urn:revet:iam:acme-corp:service-account/550e8400-e29b-41d4-a716-446655440000`
- `urn:revet:iam::service-account/550e8400-e29b-41d4-a716-446655440000` (no tenant)

## Persistence

### ServiceAccountEntity

```kotlin
package com.revethq.iam.serviceaccount.persistence.entity

@Entity
@Table(name = "revet_service_accounts")
open class ServiceAccountEntity {
    @Id lateinit var id: UUID
    @Column(nullable = false) lateinit var name: String
    @Column var description: String? = null
    @Column(name = "tenant_id") var tenantId: String? = null
    @JdbcTypeCode(SqlTypes.JSON) @Column(columnDefinition = "jsonb") var metadata: Metadata = Metadata()
    @Column(name = "created_on", nullable = false) lateinit var createdOn: OffsetDateTime
    @Column(name = "updated_on", nullable = false) lateinit var updatedOn: OffsetDateTime

    fun toDomain(): ServiceAccount
    companion object { fun fromDomain(serviceAccount: ServiceAccount): ServiceAccountEntity }
}
```

Database table: `revet_service_accounts`

### ServiceAccountRepository

```kotlin
package com.revethq.iam.serviceaccount.persistence.repository

@ApplicationScoped
class ServiceAccountRepository : PanacheRepositoryBase<ServiceAccountEntity, UUID>
```

### Pagination

```kotlin
package com.revethq.iam.serviceaccount.persistence

data class Page<T>(
    val items: List<T>,
    val totalCount: Long,
    val startIndex: Int,
    val itemsPerPage: Int
)
```

## ServiceAccountService Interface

```kotlin
package com.revethq.iam.serviceaccount.persistence.service

interface ServiceAccountService {
    fun create(serviceAccount: ServiceAccount): ServiceAccount
    fun findById(id: UUID): ServiceAccount?
    fun list(startIndex: Int, count: Int): Page<ServiceAccount>
    fun update(serviceAccount: ServiceAccount): ServiceAccount
    fun delete(id: UUID): Boolean
    fun count(): Long
}
```

### Usage

```kotlin
@Inject
lateinit var serviceAccountService: ServiceAccountService

// Create
val sa = serviceAccountService.create(
    ServiceAccount(
        id = UUID.randomUUID(),
        name = "CI Bot",
        description = "GitHub Actions service account",
        tenantId = "acme-corp"
    )
)

// Find by ID
val found = serviceAccountService.findById(sa.id)

// List with pagination
val page = serviceAccountService.list(startIndex = 0, count = 20)

// Update
val updated = serviceAccountService.update(sa.copy(name = "Updated Bot"))

// Delete
serviceAccountService.delete(sa.id)
```

## REST API Endpoints

### ServiceAccountResource

```
POST   /service-accounts               Create service account
GET    /service-accounts               List with pagination (?page=0&size=20)
GET    /service-accounts/{id}          Get by ID
PUT    /service-accounts/{id}          Update
DELETE /service-accounts/{id}          Delete
```

### ServiceAccountProfileResource

```
GET    /service-accounts/{id}/profile  Get profile
PUT    /service-accounts/{id}/profile  Set/update profile
```

Uses `ProfileType.ServiceAccount` with the existing Profile system from `user-persistence:runtime`.

### ServiceAccountPolicyResource

```
GET    /service-accounts/{id}/policies List attached policies (?page=0&size=20)
```

Uses `PolicyAttachmentService.listAttachedPoliciesForPrincipal(serviceAccount.toUrn())`.

## Request/Response DTOs

```kotlin
package com.revethq.iam.serviceaccount.web.dto

data class CreateServiceAccountRequest(
    val name: String,
    val description: String? = null,
    val tenantId: String? = null
)

data class UpdateServiceAccountRequest(
    val name: String,
    val description: String? = null,
    val tenantId: String? = null
)

data class ServiceAccountResponse(
    val id: UUID,
    val name: String,
    val description: String?,
    val tenantId: String?,
    val createdOn: OffsetDateTime?,
    val updatedOn: OffsetDateTime?
)

data class PageResponse<T>(
    val content: List<T>,
    val page: Int,
    val size: Int,
    val hasMore: Boolean
)
```

## Policy Attachment

Policies are attached to service accounts using their URN as the `principalUrn`:

```kotlin
// Attach a policy to a service account
policyAttachmentService.attach(
    policyId = policyId,
    principalUrn = serviceAccount.toUrn()  // "urn:revet:iam:acme-corp:service-account/{id}"
)

// Evaluate permissions
val result = policyEvaluator.evaluate(
    AuthorizationRequest(
        principalUrn = serviceAccount.toUrn(),
        action = "documents:Read",
        resourceUrn = "urn:revet:documents:acme-corp:document/123"
    )
)
```

## Module Architecture

Unlike `user-persistence` and `permission-persistence`, `service-account-persistence` is a flat module (not a Quarkus extension with runtime/deployment submodules). It uses Jandex indexing for CDI/JPA bean discovery. This avoids Quarkus extension Gradle plugin conflicts when multiple extension runtimes coexist in the same dependency graph.
