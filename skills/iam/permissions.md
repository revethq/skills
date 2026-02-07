# Permissions Module

Package: `com.revethq.iam.permission`

## URN (Uniform Resource Name)

```kotlin
package com.revethq.iam.permission.domain

data class Urn(
    val namespace: String,
    val service: String,
    val tenant: String,
    val resourceType: String,
    val resourceId: String
) {
    override fun toString(): String
    fun matches(pattern: Urn): Boolean
    fun matches(pattern: String): Boolean

    companion object {
        fun parse(urn: String): Urn?
        fun parseOrThrow(urn: String): Urn
    }
}
```

### Format

```
urn:{namespace}:{service}:{tenant}:{resourceType}/{resourceId}
```

### Wildcard Matching

- `*` matches single path segment (no slashes)
- `**` matches zero or more path segments (hierarchical)
- Only `resourceType` and `resourceId` support wildcards

### Examples

```kotlin
val urn = Urn.parseOrThrow("urn:revet:iam:acme-corp:user/alice")
urn.namespace     // "revet"
urn.service       // "iam"
urn.tenant        // "acme-corp"
urn.resourceType  // "user"
urn.resourceId    // "alice"

// Pattern matching
urn.matches("urn:revet:iam:acme-corp:user/*")   // true
urn.matches("urn:revet:iam:*:user/alice")       // true
```

## Policy

```kotlin
package com.revethq.iam.permission.domain

data class Policy(
    var id: UUID,
    var name: String,
    var description: String? = null,
    var version: String,
    var statements: List<Statement>,
    var tenantId: String? = null,
    var metadata: Metadata = Metadata(),
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
) {
    init {
        require(statements.isNotEmpty()) { "Policy must have at least one statement" }
    }
}
```

### Constraints

- `statements` must have at least one entry
- `name` is unique per `tenantId`
- `tenantId == null` indicates a global policy
- `version` is a string (e.g., "2026-01-15")

## Statement

```kotlin
package com.revethq.iam.permission.domain

data class Statement(
    val sid: String? = null,
    val effect: Effect,
    val actions: List<String>,
    val resources: List<String>,
    val conditions: Map<String, Map<String, List<String>>> = emptyMap()
) {
    init {
        require(actions.isNotEmpty()) { "Statement must have at least one action" }
        require(resources.isNotEmpty()) { "Statement must have at least one resource" }
    }

    fun matchesAction(action: String): Boolean
    fun matchesResource(resource: String): Boolean
    fun matchesResource(resource: Urn): Boolean
}

enum class Effect {
    ALLOW,
    DENY
}
```

### Action Format

```
{service}:{action}
```

Examples:
- `iam:CreateUser`
- `iam:*` (all IAM actions)
- `documents:Read`

### Condition Structure

```kotlin
conditions = mapOf(
    "StringEquals" to mapOf(
        "revet:SourceIp" to listOf("192.168.1.0/24")
    ),
    "DateGreaterThan" to mapOf(
        "revet:CurrentTime" to listOf("2026-01-01T00:00:00Z")
    )
)
```

## PolicyAttachment

```kotlin
package com.revethq.iam.permission.domain

data class PolicyAttachment(
    var id: UUID,
    var policyId: UUID,
    var principalUrn: String,
    var attachedOn: OffsetDateTime? = null,
    var attachedBy: String? = null
)
```

Attaches a policy to a principal (User or Group URN).

## Authorization Request/Result

```kotlin
package com.revethq.iam.permission.evaluation

data class AuthorizationRequest(
    val principalUrn: String,
    val action: String,
    val resourceUrn: String,
    val context: ConditionContext = ConditionContext()
) {
    fun toConditionContext(): ConditionContext
}

data class AuthorizationResult(
    val decision: AuthorizationDecision,
    val matchingStatements: List<MatchedStatement> = emptyList(),
    val isExplicitDeny: Boolean = false
) {
    fun isAllowed(): Boolean
    fun isDenied(): Boolean

    companion object {
        fun implicitDeny(): AuthorizationResult
        fun explicitDeny(matchingStatements: List<MatchedStatement>): AuthorizationResult
        fun allow(matchingStatements: List<MatchedStatement>): AuthorizationResult
    }
}

enum class AuthorizationDecision {
    ALLOW,
    DENY
}

data class MatchedStatement(
    val statement: Statement,
    val policyName: String
)
```

## Condition Evaluation

```kotlin
package com.revethq.iam.permission.condition

data class ConditionContext(
    val principalId: String? = null,
    val sourceIp: String? = null,
    val requestedAction: String? = null,
    val requestedResource: String? = null,
    val currentTime: OffsetDateTime = OffsetDateTime.now(),
    val customVariables: Map<String, String> = emptyMap()
) {
    fun resolveVariable(variable: String): String
    fun resolveVariables(value: String): String
    fun getValue(key: String): String?
    fun hasKey(key: String): Boolean
}
```

### Built-in Variables

| Variable | Description |
|----------|-------------|
| `${revet:PrincipalId}` | Principal URN |
| `${revet:SourceIp}` | Request source IP |
| `${revet:RequestedAction}` | Action being requested |
| `${revet:RequestedResource}` | Resource URN being accessed |
| `${revet:CurrentTime}` | Current timestamp (ISO 8601) |

### Condition Operators

```kotlin
enum class ConditionOperator {
    // String
    STRING_EQUALS, STRING_NOT_EQUALS,
    STRING_EQUALS_IGNORE_CASE, STRING_NOT_EQUALS_IGNORE_CASE,
    STRING_LIKE, STRING_NOT_LIKE,

    // Numeric
    NUMERIC_EQUALS, NUMERIC_NOT_EQUALS,
    NUMERIC_LESS_THAN, NUMERIC_LESS_THAN_EQUALS,
    NUMERIC_GREATER_THAN, NUMERIC_GREATER_THAN_EQUALS,

    // Date
    DATE_EQUALS, DATE_NOT_EQUALS,
    DATE_LESS_THAN, DATE_LESS_THAN_EQUALS,
    DATE_GREATER_THAN, DATE_GREATER_THAN_EQUALS,

    // Other
    BOOL,
    IP_ADDRESS, NOT_IP_ADDRESS,
    NULL
}
```

### Condition Logic

- Multiple operators: AND (all must pass)
- Multiple values for same key: OR (any can match)
- IP addresses support CIDR notation (e.g., `192.168.0.0/16`)

## PolicyEvaluator Interface

```kotlin
package com.revethq.iam.permission.evaluation

interface PolicyEvaluator {
    fun evaluate(request: AuthorizationRequest): AuthorizationResult
}

class DefaultPolicyEvaluator @Inject constructor(
    private val policyCollector: PolicyCollector
) : PolicyEvaluator {
    override fun evaluate(request: AuthorizationRequest): AuthorizationResult
    fun evaluateWithPolicies(request: AuthorizationRequest, policies: List<Policy>): AuthorizationResult
}
```

## PolicyCollector Interface

```kotlin
package com.revethq.iam.permission.evaluation

interface PolicyCollector {
    fun collectPolicies(principalUrn: String): List<Policy>
}
```

Default implementation: `PanachePolicyCollector` (uses Hibernate Panache).

## PolicyService Interface

```kotlin
package com.revethq.iam.permission.persistence.service

interface PolicyService {
    fun create(policy: Policy): Policy
    fun findById(id: UUID): Policy?
    fun findByName(name: String, tenantId: String? = null): Policy?
    fun list(startIndex: Int, count: Int, tenantId: String? = null): Page<Policy>
    fun update(policy: Policy): Policy
    fun delete(id: UUID): Boolean
    fun count(tenantId: String? = null): Long
}
```

## PolicyAttachmentService Interface

```kotlin
package com.revethq.iam.permission.persistence.service

interface PolicyAttachmentService {
    fun attach(policyId: UUID, principalUrn: String, attachedBy: String? = null): PolicyAttachment
    fun findById(attachmentId: UUID): PolicyAttachment?
    fun detach(policyId: UUID, attachmentId: UUID): Boolean
    fun listAttachmentsForPolicy(policyId: UUID): List<PolicyAttachment>
    fun listPoliciesForPrincipal(principalUrn: String): List<Policy>
    fun listAttachedPoliciesForPrincipal(principalUrn: String): List<AttachedPolicy>
    fun isAttached(policyId: UUID, principalUrn: String): Boolean
}

data class AttachedPolicy(
    val attachmentId: UUID,
    val policy: Policy,
    val attachedOn: OffsetDateTime?,
    val attachedBy: String?
)
```

## JAX-RS Authorization Filter

```kotlin
package com.revethq.iam.permission.web.filter

@Provider
@Priority(Priorities.AUTHORIZATION)
class AuthorizationFilter : ContainerRequestFilter {
    @Inject lateinit var policyEvaluator: PolicyEvaluator
    @Inject lateinit var authorizationContext: AuthorizationContext

    override fun filter(requestContext: ContainerRequestContext)
}

@ApplicationScoped
class AuthorizationContext {
    var principalUrn: String?
    var tenantId: String?
    var sourceIp: String?
}
```

Use `@RequiresPermission` annotation on JAX-RS endpoints:

```kotlin
@GET
@Path("/{id}")
@RequiresPermission(action = "documents:Read", resource = "urn:revet:documents:{tenantId}:document/{id}")
fun getDocument(@PathParam("id") id: UUID): Response
```

## REST API Endpoints

### PolicyResource

```
POST   /policies                          Create policy
GET    /policies/{id}                     Get policy
GET    /policies?startIndex=0&count=100   List policies
PUT    /policies/{id}                     Update policy
DELETE /policies/{id}                     Delete policy
POST   /policies/{id}/attachments         Attach to principal
DELETE /policies/{id}/attachments/{attachmentId}  Detach
GET    /policies/{id}/attachments         List attachments
```
