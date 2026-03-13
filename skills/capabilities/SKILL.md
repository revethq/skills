---
name: revet-capabilities
description: Feature-level capability evaluation - aggregates permissions into capabilities with tenant overrides and per-user resolution
---

# Revet Capabilities Library

Server-side capability evaluation for Kotlin/Quarkus applications. Services declare capabilities (feature-level groupings of permissions), and the API evaluates them for the current user. Tenant-level overrides enable admins to disable capabilities independent of user permissions.

A capability is **available** when `enabled` (not disabled by tenant override) AND `granted` (user has at least one of its permissions).

## Dependency Coordinates

**Group ID:** `com.revethq.capabilities`
**Version:** `0.1.0`

### Modules

| Artifact | Purpose |
|----------|---------|
| `revet-capabilities-core` | Domain models, SPI, evaluator, store interface (no Quarkus dependency) |
| `revet-capabilities-web` | JAX-RS REST API, DTOs |
| `revet-capabilities-persistence-runtime` | Hibernate Panache persistence for tenant overrides |

### Gradle

```kotlin
implementation("com.revethq.capabilities:revet-capabilities-core:0.1.0")
implementation("com.revethq.capabilities:revet-capabilities-web:0.1.0")
implementation("com.revethq.capabilities:revet-capabilities-persistence-runtime:0.1.0")
```

### Maven

```xml
<dependency>
    <groupId>com.revethq.capabilities</groupId>
    <artifactId>revet-capabilities-core</artifactId>
    <version>0.1.0</version>
</dependency>
```

## Core Concepts

### CapabilityDeclaration

A feature-level grouping of permissions declared by a service:

```kotlin
package com.revethq.capabilities.domain

data class CapabilityDeclaration(
    val id: String,
    val name: String,
    val description: String? = null,
    val category: String? = null,
    val permissions: List<PermissionRef>,
)
```

### PermissionRef

References a permission action and optional resource URN template:

```kotlin
package com.revethq.capabilities.domain

data class PermissionRef(
    val action: String,
    val resourceType: String? = null,
)
```

The `resourceType` supports `{tenantId}` placeholder substitution at evaluation time (e.g., `urn:revet:core:{tenantId}:organization/*`).

### CapabilityManifest

Groups capability declarations by service:

```kotlin
package com.revethq.capabilities.domain

data class CapabilityManifest(
    val service: String,
    val capabilities: List<CapabilityDeclaration>,
)
```

### TenantCapability

Tenant-level override for a capability (enable/disable):

```kotlin
package com.revethq.capabilities.domain

data class TenantCapability(
    val id: UUID? = null,
    val tenantId: String,
    val capabilityId: String,
    val enabled: Boolean,
    val reason: String? = null,
)
```

### ResolvedCapability

Result of evaluating a capability for a specific user and tenant:

```kotlin
package com.revethq.capabilities.domain

data class ResolvedCapability(
    val id: String,
    val name: String,
    val description: String? = null,
    val category: String? = null,
    val granted: Boolean,
    val enabled: Boolean,
    val permissions: List<ResolvedPermission>,
)
```

### ResolvedPermission

```kotlin
package com.revethq.capabilities.domain

data class ResolvedPermission(
    val action: String,
    val granted: Boolean,
)
```

## Capability Discovery (SPI)

### CapabilityProvider

Services implement this interface to declare their capabilities:

```kotlin
package com.revethq.capabilities.discovery

interface CapabilityProvider {
    fun manifest(): CapabilityManifest
}
```

### CapabilityRegistry

Collects all `CapabilityProvider` implementations via CDI:

```kotlin
package com.revethq.capabilities.discovery

@ApplicationScoped
class CapabilityRegistry @Inject constructor(
    private val providers: Instance<CapabilityProvider>,
) {
    fun allManifests(): List<CapabilityManifest>
    fun allCapabilities(): List<CapabilityDeclaration>
}
```

### Example Provider

```kotlin
@ApplicationScoped
class CoreCapabilityProvider : CapabilityProvider {
    override fun manifest() = CapabilityManifest(
        service = "core",
        capabilities = listOf(
            CapabilityDeclaration(
                id = "core:manage-organizations",
                name = "Manage Organizations",
                description = "Create, update, and delete organizations",
                category = "Organizations",
                permissions = listOf(
                    PermissionRef("core:CreateOrganization", "urn:revet:core:{tenantId}:organization/*"),
                    PermissionRef("core:UpdateOrganization", "urn:revet:core:{tenantId}:organization/*"),
                    PermissionRef("core:DeleteOrganization", "urn:revet:core:{tenantId}:organization/*"),
                ),
            ),
        ),
    )
}
```

## Evaluation

### CapabilityEvaluator

```kotlin
package com.revethq.capabilities.evaluation

interface CapabilityEvaluator {
    fun evaluate(principalUrn: String, tenantId: String): List<ResolvedCapability>
}
```

`DefaultCapabilityEvaluator` is the `@ApplicationScoped` implementation. It:

1. Collects all capability declarations from the `CapabilityRegistry`
2. Loads tenant overrides from `TenantCapabilityStore`
3. Collects policies **once** via `PolicyCollector.collectPolicies(principalUrn)`
4. Evaluates each permission using `PolicyEvaluator.evaluateWithPolicies()` — avoids N database round-trips
5. Replaces `{tenantId}` in `PermissionRef.resourceType` templates before evaluation

**Decision rules:**
- `granted = true` if the user has **any** of the capability's permissions (OR semantics)
- `enabled = true` by default, unless a `TenantCapability` override sets it to `false`
- `available = enabled && granted` (computed in the DTO layer)

### TenantCapabilityStore

Port interface for persistence of tenant overrides:

```kotlin
package com.revethq.capabilities.service

interface TenantCapabilityStore {
    fun findByTenantId(tenantId: String): List<TenantCapability>
    fun findByTenantIdAndCapabilityId(tenantId: String, capabilityId: String): TenantCapability?
    fun save(tenantCapability: TenantCapability): TenantCapability
    fun delete(tenantId: String, capabilityId: String): Boolean
}
```

`PanacheTenantCapabilityStore` in the persistence-runtime module implements this with an upsert-on-save pattern.

## REST API

Base path: `/api/v1/capabilities`

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Evaluate all capabilities for the current user |

Principal and tenant are resolved from `AuthorizationContext` (provided by `revet-permission-web`).

### Response Shape

```json
{
  "capabilities": [
    {
      "id": "core:manage-organizations",
      "name": "Manage Organizations",
      "description": "Create, update, and delete organizations",
      "category": "Organizations",
      "granted": true,
      "enabled": true,
      "available": true,
      "permissions": [
        { "action": "core:CreateOrganization", "granted": true },
        { "action": "core:UpdateOrganization", "granted": true },
        { "action": "core:DeleteOrganization", "granted": false }
      ]
    }
  ]
}
```

### DTOs

```kotlin
package com.revethq.capabilities.web.dto

data class ResolvedCapabilityDto(
    val id: String,
    val name: String,
    val description: String?,
    val category: String?,
    val granted: Boolean,
    val enabled: Boolean,
    val available: Boolean,
    val permissions: List<ResolvedPermissionDto>,
)

data class ResolvedPermissionDto(
    val action: String,
    val granted: Boolean,
)

data class CapabilityListResponse(
    val capabilities: List<ResolvedCapabilityDto>,
)
```

## Extension Points

### CapabilityProvider

Implement to declare capabilities for your service:

```kotlin
@ApplicationScoped
class MyServiceCapabilityProvider : CapabilityProvider {
    override fun manifest() = CapabilityManifest(
        service = "my-service",
        capabilities = listOf(
            CapabilityDeclaration(
                id = "my-service:feature-name",
                name = "Feature Name",
                description = "What this feature does",
                category = "My Category",
                permissions = listOf(
                    PermissionRef("my-service:DoSomething", "urn:revet:my-service:{tenantId}:resource/*"),
                ),
            ),
        ),
    )
}
```

Providers are discovered via CDI `Instance<CapabilityProvider>` injection. Add a `CapabilityProvider` bean and it will be picked up automatically by the `CapabilityRegistry`.

## Persistence

### Database Table

```sql
CREATE TABLE revet_tenant_capabilities (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR(255) NOT NULL,
    capability_id VARCHAR(255) NOT NULL,
    enabled BOOLEAN NOT NULL DEFAULT true,
    reason TEXT,
    created_on TIMESTAMPTZ NOT NULL,
    updated_on TIMESTAMPTZ NOT NULL,
    CONSTRAINT uq_revet_tenant_capabilities_tenant_capability UNIQUE (tenant_id, capability_id)
);
```

Flyway migration: `V1__create_tenant_capabilities_table.sql`

## Key Constraints

- Capability IDs should follow the format `{service}:{feature-name}` (e.g., `core:manage-organizations`)
- `PermissionRef.action` must match an action declared by a `PermissionProvider`
- `PermissionRef.resourceType` supports `{tenantId}` placeholder — replaced at evaluation time
- A capability with no `TenantCapability` override is enabled by default
- `granted` uses OR semantics — any single granted permission makes the capability granted
- The evaluator collects IAM policies once per evaluation call, not once per permission check
- `available` is a computed field (`enabled && granted`), not stored
