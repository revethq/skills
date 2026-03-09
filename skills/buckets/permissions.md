# Permissions Module

Package: `com.revethq.buckets.permission`

## Actions

```kotlin
package com.revethq.buckets.permission

object Actions {
    const val SERVICE = "buckets"

    object Bucket {
        const val LIST = "buckets:ListBuckets"
        const val GET = "buckets:GetBucket"
        const val CREATE = "buckets:CreateBucket"
        const val UPDATE = "buckets:UpdateBucket"
        const val DELETE = "buckets:DeleteBucket"

        val ALL = listOf(LIST, GET, CREATE, UPDATE, DELETE)
        val READ_ONLY = listOf(LIST, GET)
        val WRITE = listOf(CREATE, UPDATE)
    }

    const val ALL_ACTIONS = "buckets:*"
    const val GLOBAL_WILDCARD = "*"
}
```

### Action Groups

- `Actions.Bucket.ALL` - All bucket operations
- `Actions.Bucket.READ_ONLY` - List and Get only
- `Actions.Bucket.WRITE` - Create and Update only

## BucketsUrn

```kotlin
package com.revethq.buckets.permission

object BucketsUrn {
    const val NAMESPACE = "revet"
    const val SERVICE = "buckets"

    object ResourceType {
        const val BUCKET = "bucket"
    }

    fun build(tenant: String, resourceType: String, resourceId: String): String
    fun build(tenant: String, resourceType: String, resourceId: UUID): String
    fun wildcard(tenant: String, resourceType: String): String
    fun bucket(tenant: String, id: UUID): String
    fun bucket(tenant: String, id: String): String
    fun bucketWildcard(tenant: String): String
}
```

### URN Format

```
urn:revet:buckets:{tenantId}:bucket/{bucketUuid}
```

### Examples

```kotlin
// Single bucket
BucketsUrn.bucket("acme-corp", uuid)
// → "urn:revet:buckets:acme-corp:bucket/550e8400-e29b-41d4-a716-446655440000"

// All buckets in tenant
BucketsUrn.bucketWildcard("acme-corp")
// → "urn:revet:buckets:acme-corp:bucket/*"

// Generic wildcard
BucketsUrn.wildcard("acme-corp", "bucket")
// → "urn:revet:buckets:acme-corp:bucket/*"
```

## REST Endpoint Permissions

Each endpoint is protected with `@RequiresPermission`:

```kotlin
@GET
@RequiresPermission(action = Actions.Bucket.LIST, resource = "urn:revet:buckets:{tenantId}:bucket/*")
fun listBuckets(): List<BucketDTO>

@GET
@Path("/{uuid}")
@RequiresPermission(action = Actions.Bucket.GET, resource = "urn:revet:buckets:{tenantId}:bucket/{uuid}")
fun getBucket(@PathParam("uuid") uuid: UUID): Response

@POST
@RequiresPermission(action = Actions.Bucket.CREATE, resource = "urn:revet:buckets:{tenantId}:bucket/*")
fun createBucket(request: CreateBucketRequest): Response

@PUT
@Path("/{uuid}")
@RequiresPermission(action = Actions.Bucket.UPDATE, resource = "urn:revet:buckets:{tenantId}:bucket/{uuid}")
fun updateBucket(@PathParam("uuid") uuid: UUID, request: UpdateBucketRequest): Response

@DELETE
@Path("/{uuid}")
@RequiresPermission(action = Actions.Bucket.DELETE, resource = "urn:revet:buckets:{tenantId}:bucket/{uuid}")
fun deleteBucket(@PathParam("uuid") uuid: UUID): Response
```

## IAM Policy Examples

### Full access to all buckets

```kotlin
val fullAccess = Policy(
    id = UUID.randomUUID(),
    name = "buckets-full-access",
    version = "2026-01-15",
    statements = listOf(
        Statement(
            effect = Effect.ALLOW,
            actions = listOf(Actions.ALL_ACTIONS),
            resources = listOf(BucketsUrn.bucketWildcard("acme-corp"))
        )
    ),
    tenantId = "acme-corp"
)
```

### Read-only access

```kotlin
val readOnly = Policy(
    id = UUID.randomUUID(),
    name = "buckets-read-only",
    version = "2026-01-15",
    statements = listOf(
        Statement(
            effect = Effect.ALLOW,
            actions = Actions.Bucket.READ_ONLY,
            resources = listOf(BucketsUrn.bucketWildcard("acme-corp"))
        )
    ),
    tenantId = "acme-corp"
)
```

### Single bucket access

```kotlin
val singleBucket = Policy(
    id = UUID.randomUUID(),
    name = "production-bucket-access",
    version = "2026-01-15",
    statements = listOf(
        Statement(
            effect = Effect.ALLOW,
            actions = Actions.Bucket.ALL,
            resources = listOf(BucketsUrn.bucket("acme-corp", bucketUuid))
        )
    ),
    tenantId = "acme-corp"
)
```
