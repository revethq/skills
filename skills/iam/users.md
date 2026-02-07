# Users Module

Package: `com.revethq.iam.user`

## Domain Models

### User

```kotlin
package com.revethq.iam.user.domain

data class User(
    var id: UUID,
    var username: String,
    var email: String,
    var metadata: Metadata = Metadata(),
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
)
```

### Constraints

- `username` is globally unique
- `email` is globally unique

### Group

```kotlin
package com.revethq.iam.user.domain

data class Group(
    var id: UUID,
    var displayName: String,
    var externalId: String? = null,
    var metadata: Metadata = Metadata(),
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
)
```

### GroupMember

```kotlin
package com.revethq.iam.user.domain

data class GroupMember(
    var id: UUID? = null,
    var groupId: UUID,
    var memberId: UUID,
    var memberType: MemberType = MemberType.USER,
    var createdOn: OffsetDateTime? = null
)

enum class MemberType {
    USER,
    GROUP
}
```

Groups can contain both users and nested groups via `MemberType`.

### Profile

```kotlin
package com.revethq.iam.user.domain

data class Profile(
    var id: UUID? = null,
    var resource: UUID? = null,
    var profileType: ProfileType? = null,
    var profile: Map<String, Any>? = null,
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
)

enum class ProfileType {
    User,
    Application
}
```

Extensible profile data stored as JSON map.

### IdentityProvider

```kotlin
package com.revethq.iam.user.domain

data class IdentityProvider(
    var id: UUID,
    var name: String,
    var externalId: String? = null,
    var metadata: Metadata = Metadata(),
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
)
```

### IdentityProviderLink

```kotlin
package com.revethq.iam.user.domain

data class IdentityProviderLink(
    var id: UUID,
    var userId: UUID,
    var identityProviderId: UUID,
    var externalId: String,
    var metadata: Metadata = Metadata(),
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
)
```

Maps external IdP user IDs to internal users. The `externalId` is unique per `identityProviderId`.

## Pagination

```kotlin
package com.revethq.iam.user.persistence

data class Page<T>(
    val items: List<T>,
    val totalCount: Long,
    val startIndex: Int,
    val itemsPerPage: Int
)
```

## UserService Interface

```kotlin
package com.revethq.iam.user.persistence.service

interface UserService {
    fun create(user: User, identityProviderId: UUID, externalId: String?): User
    fun findById(id: UUID): User?
    fun findByUsername(username: String): User?
    fun findByEmail(email: String): User?
    fun findByExternalId(externalId: String, identityProviderId: UUID): User?
    fun getExternalId(userId: UUID, identityProviderId: UUID): String?
    fun list(startIndex: Int, count: Int): Page<User>
    fun update(user: User): User
    fun updateExternalId(userId: UUID, identityProviderId: UUID, externalId: String?)
    fun delete(id: UUID): Boolean
    fun count(): Long
}
```

### Usage

```kotlin
@Inject
lateinit var userService: UserService

// Create user with identity provider link
val user = userService.create(
    user = User(
        id = UUID.randomUUID(),
        username = "alice",
        email = "alice@example.com"
    ),
    identityProviderId = idpId,
    externalId = "external-123"
)

// Find by various criteria
val byId = userService.findById(user.id)
val byUsername = userService.findByUsername("alice")
val byEmail = userService.findByEmail("alice@example.com")
val byExternal = userService.findByExternalId("external-123", idpId)

// Pagination
val page = userService.list(startIndex = 0, count = 20)
```

## GroupService Interface

```kotlin
package com.revethq.iam.user.persistence.service

interface GroupService {
    fun create(group: Group): Group
    fun findById(id: UUID): Group?
    fun findByExternalId(externalId: String): Group?
    fun findByDisplayName(displayName: String): Group?
    fun list(startIndex: Int, count: Int): Page<Group>
    fun update(group: Group): Group
    fun delete(id: UUID): Boolean
    fun count(): Long
    fun getMembers(groupId: UUID): List<GroupMember>
    fun addMember(groupId: UUID, member: GroupMember): GroupMember
    fun removeMember(groupId: UUID, memberId: UUID): Boolean
    fun setMembers(groupId: UUID, members: List<GroupMember>): List<GroupMember>
}
```

### Usage

```kotlin
@Inject
lateinit var groupService: GroupService

// Create group
val group = groupService.create(
    Group(
        id = UUID.randomUUID(),
        displayName = "Administrators"
    )
)

// Add members
groupService.addMember(
    groupId = group.id,
    member = GroupMember(
        groupId = group.id,
        memberId = userId,
        memberType = MemberType.USER
    )
)

// Nested groups
groupService.addMember(
    groupId = parentGroup.id,
    member = GroupMember(
        groupId = parentGroup.id,
        memberId = childGroup.id,
        memberType = MemberType.GROUP
    )
)

// List members
val members = groupService.getMembers(group.id)
```

## Repository Classes

```kotlin
package com.revethq.iam.user.persistence.repository

@ApplicationScoped
class UserRepository : PanacheRepositoryBase<UserEntity, UUID> {
    fun findByUsername(username: String): UserEntity?
    fun findByEmail(email: String): UserEntity?
}

@ApplicationScoped
class GroupRepository : PanacheRepositoryBase<GroupEntity, UUID> {
    fun findByExternalId(externalId: String): GroupEntity?
    fun findByDisplayName(displayName: String): GroupEntity?
}
```

## REST API Endpoints

### UserResource

```
POST   /users                    Create user
GET    /users/{id}               Get user
GET    /users?page=0&size=20     List users
PUT    /users/{id}               Update user
DELETE /users/{id}               Delete user
```

### Request/Response DTOs

```kotlin
// CreateUserRequest
data class CreateUserRequest(
    val username: String,
    val email: String,
    val metadata: Metadata? = null
)

// UserResponse
data class UserResponse(
    val id: UUID,
    val username: String,
    val email: String,
    val metadata: Metadata,
    val createdOn: OffsetDateTime?,
    val updatedOn: OffsetDateTime?
)
```

### Headers

- `X-Identity-Provider-Id` - Identity provider UUID for linking external users
