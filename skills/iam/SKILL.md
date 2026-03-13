---
name: revet-iam
description: Integration guide for Revet IAM library - identity, access management, and permission evaluation
---

# Revet IAM Library

Identity and access management for Kotlin/Quarkus applications. Provides permission evaluation, user/group management, service accounts, and SCIM 2.0 provisioning.

## Dependency Coordinates

**Group ID:** `com.revethq.iam`
**Version:** `0.1.17`

### Modules

| Artifact | Purpose |
|----------|---------|
| `revet-permission` | Policy/Statement model, URN parsing, policy evaluation, permission discovery |
| `revet-permission-web` | JAX-RS REST API for policy management, `/.well-known/revet-permissions` discovery endpoint |
| `revet-permission-persistence-runtime` | Hibernate Panache persistence for policies |
| `revet-user` | User/Group/Profile domain models |
| `revet-user-web` | JAX-RS REST API for user/group management |
| `revet-user-persistence-runtime` | Hibernate Panache persistence for users/groups |
| `revet-service-account` | ServiceAccount domain model |
| `revet-service-account-persistence` | Hibernate Panache persistence for service accounts |
| `revet-service-account-web` | JAX-RS REST API for service account management |
| `revet-scim` | SCIM 2.0 User/Group provisioning endpoints |

### Gradle

```kotlin
implementation("com.revethq.iam:revet-permission:0.1.17")
implementation("com.revethq.iam:revet-user:0.1.17")
implementation("com.revethq.iam:revet-service-account:0.1.17")
implementation("com.revethq.iam:revet-scim:0.1.17")
```

### Maven

```xml
<dependency>
    <groupId>com.revethq.iam</groupId>
    <artifactId>revet-permission</artifactId>
    <version>0.1.17</version>
</dependency>
```

## Core Concepts

### URN Format

Resources are identified by URNs:
```
urn:{namespace}:{service}:{tenant}:{resourceType}/{resourceId}
```

Examples:
- `urn:revet:iam:acme-corp:user/alice`
- `urn:revet:iam:acme-corp:service-account/550e8400-e29b-41d4-a716-446655440000`

Components:
- `namespace` - Organization namespace (e.g., `revet`)
- `service` - Service identifier (e.g., `iam`, `documents`)
- `tenant` - Tenant/organization identifier (empty for global)
- `resourceType` - Resource category (e.g., `user`, `group`, `policy`)
- `resourceId` - Unique resource identifier

### Policy Model

Policies contain statements that grant or deny permissions:

```kotlin
val policy = Policy(
    id = UUID.randomUUID(),
    name = "user-management-policy",
    version = "2026-01-15",
    statements = listOf(
        Statement(
            effect = Effect.ALLOW,
            actions = listOf("iam:CreateUser", "iam:UpdateUser"),
            resources = listOf("urn:revet:iam:acme-corp:user/*")
        )
    ),
    tenantId = "acme-corp"
)
```

### Authorization Decision Rules

1. **Explicit DENY** - Any matching Deny statement → DENY (highest precedence)
2. **Allow** - Any matching Allow statement (no Deny) → ALLOW
3. **Implicit DENY** - No statements match → DENY (default)

## Related Documentation

- [permissions.md](./permissions.md) - URN format, Policy/Statement classes, condition evaluation
- [users.md](./users.md) - User/Group/Profile data classes, service interfaces
- [service-accounts.md](./service-accounts.md) - ServiceAccount domain, persistence, REST API
- [scim.md](./scim.md) - SCIM 2.0 DTOs, endpoint contracts, filter grammar

## Permission Discovery

Downstream applications can advertise all permissions they understand at a well-known endpoint. This aggregates permissions declared by the application itself and all of its library dependencies.

### Declaring Permissions

Implement `PermissionProvider` as an `@ApplicationScoped` CDI bean to declare your service's permissions. The `revet-permission` module contains the interface, so any module can implement it without depending on `revet-permission-web`.

```kotlin
import com.revethq.iam.permission.discovery.PermissionDeclaration
import com.revethq.iam.permission.discovery.PermissionManifest
import com.revethq.iam.permission.discovery.PermissionProvider
import jakarta.enterprise.context.ApplicationScoped

@ApplicationScoped
class BillingPermissionProvider : PermissionProvider {
    override fun manifest() = PermissionManifest(
        service = "billing",
        permissions = listOf(
            PermissionDeclaration(
                action = "billing:CreateInvoice",
                description = "Create an invoice",
                resourceType = "urn:revet:billing:{tenantId}:invoice/{invoiceId}",
            ),
            PermissionDeclaration(
                action = "billing:GetInvoice",
                description = "Retrieve an invoice by ID",
                resourceType = "urn:revet:billing:{tenantId}:invoice/{invoiceId}",
            ),
        ),
    )
}
```

Each `PermissionDeclaration` has:
- `action` (required) — follows the `{service}:{action}` format (e.g., `billing:CreateInvoice`)
- `description` (optional) — human-readable explanation
- `resourceType` (optional) — URN template showing the applicable resource pattern

### How Aggregation Works

When the application starts, CDI discovers all `PermissionProvider` beans on the classpath — including those from transitive library dependencies (via Jandex indexes). The `PermissionRegistry` collects them automatically. No manual registration is needed.

For example, if your app depends on `revet-permission-web` (which brings in IAM permissions) and also declares its own `BillingPermissionProvider`, both sets of permissions are aggregated.

### Discovery Endpoint

Applications that include `revet-permission-web` automatically expose:

```
GET /.well-known/revet-permissions
```

This returns all aggregated permissions grouped by service:

```json
{
  "manifests": [
    {
      "service": "iam",
      "permissions": [
        {
          "action": "iam:CreateUser",
          "description": "Create a new user",
          "resourceType": "urn:revet:iam:{tenantId}:user/{userId}"
        }
      ]
    },
    {
      "service": "billing",
      "permissions": [
        {
          "action": "billing:CreateInvoice",
          "description": "Create an invoice",
          "resourceType": "urn:revet:billing:{tenantId}:invoice/{invoiceId}"
        }
      ]
    }
  ]
}
```

### Programmatic Access

Inject `PermissionRegistry` to access declared permissions in code:

```kotlin
@Inject
lateinit var permissionRegistry: PermissionRegistry

// All manifests grouped by service
val manifests = permissionRegistry.allManifests()

// Flat list of all permission declarations
val allPermissions = permissionRegistry.allPermissions()
```

## Extension Points

### PolicyCollector

Implement to customize policy retrieval:

```kotlin
@ApplicationScoped
class CustomPolicyCollector : PolicyCollector {
    override fun collectPolicies(principalUrn: String): List<Policy> {
        // Fetch from external IAM, apply caching, filter by tenant
    }
}
```

### PolicyEvaluator

Implement for custom authorization logic:

```kotlin
@ApplicationScoped
class CustomPolicyEvaluator : PolicyEvaluator {
    override fun evaluate(request: AuthorizationRequest): AuthorizationResult {
        // Custom ABAC, audit logging, external service integration
    }
}
```

## Key Constraints

- Policies must have at least one statement
- Statements must have at least one action and one resource
- Policy names are unique per tenant
- `tenantId == null` indicates global policies
- Wildcard `*` matches single path segment; `**` matches hierarchical paths
- Action format: `{service}:{action}` (e.g., `iam:CreateUser`)
