# SCIM Module

Package: `com.revethq.iam.scim`

SCIM 2.0 (System for Cross-domain Identity Management) provisioning for users and groups.

## SCIM DTOs

### ScimUser

```kotlin
package com.revethq.iam.scim.dtos

@RegisterForReflection
data class ScimUser(
    val schemas: List<String> = listOf(SCHEMA_USER),
    val id: String? = null,
    val externalId: String? = null,
    val meta: ScimMeta? = null,
    val userName: String,
    val name: ScimName? = null,
    val displayName: String? = null,
    val emails: List<ScimEmail>? = null,
    val active: Boolean = true,
    val locale: String? = null,
    val password: String? = null
) {
    companion object {
        const val SCHEMA_USER = "urn:ietf:params:scim:schemas:core:2.0:User"
    }
}
```

### ScimName

```kotlin
data class ScimName(
    val formatted: String? = null,
    val familyName: String? = null,
    val givenName: String? = null,
    val middleName: String? = null,
    val honorificPrefix: String? = null,
    val honorificSuffix: String? = null
)
```

### ScimEmail

```kotlin
data class ScimEmail(
    val value: String,
    val type: String? = null,
    val primary: Boolean = false
)
```

### ScimMeta

```kotlin
data class ScimMeta(
    val resourceType: String? = null,
    val created: OffsetDateTime? = null,
    val lastModified: OffsetDateTime? = null,
    val location: String? = null,
    val version: String? = null
)
```

### ScimListResponse

```kotlin
data class ScimListResponse<T>(
    val schemas: List<String> = listOf("urn:ietf:params:scim:api:messages:2.0:ListResponse"),
    val totalResults: Long,
    val startIndex: Int,
    val itemsPerPage: Int,
    val Resources: List<T>
)
```

### ScimPatchOp

```kotlin
data class ScimPatchOp(
    val schemas: List<String> = listOf("urn:ietf:params:scim:api:messages:2.0:PatchOp"),
    val Operations: List<PatchOperation>
)

data class PatchOperation(
    val op: String,      // "add", "remove", "replace"
    val path: String?,
    val value: Any?
)
```

## SCIM Filter Grammar

```kotlin
package com.revethq.iam.scim.filter

sealed class ScimFilter {
    abstract fun matches(attributes: Map<String, Any?>): Boolean
}

data class EqFilter(val attribute: String, val value: String) : ScimFilter()
data class CoFilter(val attribute: String, val value: String) : ScimFilter()
data class SwFilter(val attribute: String, val value: String) : ScimFilter()
data class AndFilter(val left: ScimFilter, val right: ScimFilter) : ScimFilter()
data class OrFilter(val left: ScimFilter, val right: ScimFilter) : ScimFilter()
```

### Supported Filter Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `eq` | Equals | `userName eq "alice"` |
| `co` | Contains | `email co "@example.com"` |
| `sw` | Starts with | `userName sw "admin"` |
| `and` | Logical AND | `userName eq "alice" and active eq "true"` |
| `or` | Logical OR | `email eq "a@b.com" or email eq "c@d.com"` |

## REST API Endpoints

### Users (`/scim/v2/Users`)

```
POST   /scim/v2/Users              Create user
GET    /scim/v2/Users              List users (with filter)
GET    /scim/v2/Users/{id}         Get user
PUT    /scim/v2/Users/{id}         Replace user
PATCH  /scim/v2/Users/{id}         Patch user
DELETE /scim/v2/Users/{id}         Delete user
```

### UserResource

```kotlin
package com.revethq.iam.scim.api

@ScimEndpoint
@Path("/scim/v2/Users")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
class UserResource {
    @POST
    @Transactional
    fun createUser(@RequestBody scimUser: ScimUser): Response

    @GET
    fun listUsers(
        @QueryParam("filter") filter: String?,
        @QueryParam("startIndex") @DefaultValue("1") startIndex: Int,
        @QueryParam("count") @DefaultValue("100") count: Int
    ): ScimListResponse<ScimUser>

    @GET
    @Path("/{id}")
    fun getUser(@PathParam("id") id: String): ScimUser

    @PUT
    @Path("/{id}")
    @Transactional
    fun replaceUser(@PathParam("id") id: String, scimUser: ScimUser): ScimUser

    @PATCH
    @Path("/{id}")
    @Transactional
    fun patchUser(@PathParam("id") id: String, patchOp: ScimPatchOp): ScimUser

    @DELETE
    @Path("/{id}")
    @Transactional
    fun deleteUser(@PathParam("id") id: String): Response
}
```

## SCIM Conventions

### Pagination

SCIM uses 1-based indexing:
- `startIndex=1` returns the first page
- `count` specifies items per page

### Schemas

All SCIM resources include a `schemas` array:
- User: `["urn:ietf:params:scim:schemas:core:2.0:User"]`
- Group: `["urn:ietf:params:scim:schemas:core:2.0:Group"]`
- ListResponse: `["urn:ietf:params:scim:api:messages:2.0:ListResponse"]`
- PatchOp: `["urn:ietf:params:scim:api:messages:2.0:PatchOp"]`

### Default Values

- `ScimUser.active` defaults to `true`
- `ScimUser.schemas` defaults to core user schema

## Integration with User Module

The SCIM module maps to the User domain:

| SCIM Field | User Field |
|------------|------------|
| `userName` | `username` |
| `emails[primary=true].value` | `email` |
| `externalId` | `IdentityProviderLink.externalId` |
| `id` | `User.id` (UUID) |
| `active` | (stored in metadata) |

## Error Responses

SCIM errors follow RFC 7644:

```json
{
    "schemas": ["urn:ietf:params:scim:api:messages:2.0:Error"],
    "status": "404",
    "scimType": "invalidValue",
    "detail": "User not found"
}
```

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 201 | Created |
| 200 | OK (GET, PUT, PATCH) |
| 204 | No Content (DELETE) |
| 400 | Bad Request |
| 404 | Not Found |
| 409 | Conflict (duplicate) |
