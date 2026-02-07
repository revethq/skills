---
name: revet-auth
description: OAuth 2.1 / OIDC authorization server for Kotlin/Quarkus applications (in development)
---

# Revet Auth Library

OAuth 2.1 and OpenID Connect authorization server implementation for Kotlin/Quarkus applications.

**Status:** In development. APIs may change.

## Dependency Coordinates

**Group ID:** `com.revethq.auth`
**Version:** `0.0.1-SNAPSHOT`

### Modules

| Artifact | Purpose |
|----------|---------|
| `core` | Domain models, services, DTOs |
| `web` | Quarkus REST endpoints, OAuth flows |
| `persistence` | Data access layer |

### Gradle

```kotlin
implementation("com.revethq.auth:core:0.0.1-SNAPSHOT")
implementation("com.revethq.auth:web:0.0.1-SNAPSHOT")
```

## OAuth 2.1 Flows

### Authorization Code Flow (with PKCE)

```
1. GET  /{authServerId}/authorization/
   ?client_id=...&redirect_uri=...&scope=...
   &code_challenge=...&code_challenge_method=S256

2. POST /{authServerId}/authorization/
   (user submits credentials)

3. Redirect to client with code

4. POST /{authServerId}/token/
   grant_type=authorization_code
   &code=...&redirect_uri=...&code_verifier=...

5. Returns access_token, id_token, refresh_token
```

### Client Credentials Flow

```
POST /{authServerId}/token/
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=...
&client_secret=...
&scope=...
```

### Refresh Token Flow

```
POST /{authServerId}/token/
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=...
&client_id=...
```

## Core Domain Models

### AuthorizationServer

```kotlin
data class AuthorizationServer(
    var id: UUID? = null,
    var name: String? = null,
    var serverUrl: URL? = null,
    var audience: String? = null,
    var clientCredentialsTokenExpiration: Long = 3600L,
    var authorizationCodeTokenExpiration: Long = 3600L,
    var metadata: Metadata? = null,
    var scopes: List<Scope>? = null,
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
)
```

### Client

```kotlin
data class Client(
    var id: UUID? = null,
    var name: String? = null,
    var authorizationServerId: UUID? = null,
    var redirectUris: List<URI>? = null,
    var scopes: List<Scope>? = null,
    var metadata: Metadata? = null,
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
)
```

### Application (Machine-to-Machine)

```kotlin
data class Application(
    var id: UUID? = null,
    var clientId: String? = null,
    var name: String? = null,
    var authorizationServerId: UUID? = null,
    var scopes: List<Scope>? = null,
    var metadata: Metadata? = null,
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
)
```

### Scope

```kotlin
data class Scope(
    var id: UUID? = null,
    var authorizationServer: AuthorizationServer? = null,
    var name: String? = null,
    var metadata: Metadata? = null,
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null
)
```

## OpenID Connect Endpoints

### Discovery

```
GET /{authServerId}/.well-known/openid-configuration
```

Returns:
- `issuer`
- `authorization_endpoint`
- `token_endpoint`
- `userinfo_endpoint`
- `jwks_uri`
- `scopes_supported`
- `response_types_supported`
- `grant_types_supported`

### JWKS

```
GET /{authServerId}/jwks/
```

Returns JSON Web Key Set for token verification. Algorithm: RS256.

### UserInfo

```
GET /{authServerId}/userinfo/
Authorization: Bearer {access_token}
```

## Token Response

```kotlin
data class AccessTokenResponse(
    val accessToken: String?,
    val tokenType: String?,      // "Bearer"
    val expiresIn: Int?,
    val refreshToken: String?,
    val scope: String?,
    val idToken: String?         // OIDC flows
)
```

## Service Interfaces

### AuthorizationServerService

```kotlin
interface AuthorizationServerService {
    fun getAuthorizationServer(id: UUID): AuthorizationServer
    fun createAuthorizationServer(server: AuthorizationServer): AuthorizationServer
    fun getSigningKeysForAuthorizationServer(id: UUID): SigningKey
    fun validateJwtForAuthorizationServer(id: UUID, jwt: String): Map<String, Any>
    fun getJwksForAuthorizationServer(id: UUID): List<JWK>

    fun generateClientCredentialsAccessToken(
        authorizationServerId: UUID,
        applicationId: UUID,
        subject: String,
        scopes: List<Scope>,
        expiresInSeconds: Long
    ): AccessToken

    fun generateAuthorizationCodeFlowAccessToken(
        authorizationServerId: UUID,
        userId: UUID,
        subject: String,
        clientId: String,
        scopes: List<Scope>,
        expiresInSeconds: Long,
        nonce: String?
    ): AccessToken
}
```

### UserService

```kotlin
interface UserService {
    fun createUser(pair: Pair<User, Profile>): Pair<User, Profile>
    fun getUser(userId: UUID): Pair<User, Profile>
    fun getUser(username: String): Pair<User, Profile>
    fun setPassword(user: User, password: String)
    fun validatePassword(userId: UUID, password: String): Boolean
}
```

### ApplicationService

```kotlin
interface ApplicationService {
    fun createApplication(app: Application, profile: Profile): Pair<Application, Profile>
    fun getApplication(id: UUID): Pair<Application, Profile>
    fun createApplicationSecret(secret: ApplicationSecret): ApplicationSecret
    fun isApplicationSecretValid(
        authorizationServerId: UUID,
        secretId: UUID,
        secret: String
    ): Boolean
}
```

## Identity Provider Federation

```kotlin
data class IdentityProvider(
    var id: UUID? = null,
    var name: String? = null,
    var authorizationServerId: UUID? = null,
    var discoveryMethod: DiscoveryMethodEnum? = null,
    var discoveryEndpoint: String? = null,
    var wellKnownEndpoints: WellKnownEndpoints? = null,
    var clientId: String? = null,
    var clientSecret: String? = null,
    var usePkce: Boolean? = null,
    var metadata: Metadata? = null
)

enum class DiscoveryMethodEnum {
    MANUAL,
    WELL_KNOWN
}
```

## Management API Endpoints

```
POST   /applications                Create application
GET    /applications/{id}           Get application
DELETE /applications/{id}           Delete application

POST   /clients                     Create client
GET    /clients/{id}                Get client
DELETE /clients/{id}                Delete client

POST   /users                       Create user
GET    /users/{id}                  Get user
PUT    /users/{id}                  Update user
DELETE /users/{id}                  Delete user

POST   /scopes                      Create scope
GET    /scopes                      List scopes
DELETE /scopes/{id}                 Delete scope
```

## Constraints

- Token signing: RS256 algorithm
- PKCE required for authorization code flow
- Client secrets are hashed, never stored plaintext
- Token expiration: configurable per authorization server
