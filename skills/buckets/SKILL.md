---
name: revet-buckets
description: Cloud storage bucket management - multi-provider presigned URLs, credential encryption, and file operations
---

# Revet Buckets Library

Cloud storage bucket configuration and file operations for Kotlin/Quarkus applications. Manages bucket credentials with AES-256-GCM encryption, generates presigned URLs, and abstracts across S3, GCS, MinIO, and Azure Blob Storage.

## Dependency Coordinates

**Group ID:** `com.revethq.buckets`
**Version:** `0.1.0`

### Modules

| Artifact | Purpose |
|----------|---------|
| `revet-buckets-core` | Domain models, service interfaces, permission definitions (no Quarkus dependency) |
| `revet-buckets-web` | JAX-RS REST API, service implementations, Quarkus application |
| `revet-buckets-persistence-runtime` | Hibernate Panache persistence, AES encryption |
| `revet-buckets-provider-s3` | AWS S3 and MinIO storage provider |
| `revet-buckets-provider-gcs` | Google Cloud Storage provider |
| `revet-buckets-provider-azure-blob` | Azure Blob Storage provider (stub) |

### Gradle

```kotlin
implementation("com.revethq.buckets:revet-buckets-core:0.1.0")
implementation("com.revethq.buckets:revet-buckets-web:0.1.0")
implementation("com.revethq.buckets:revet-buckets-persistence-runtime:0.1.0")
implementation("com.revethq.buckets:revet-buckets-provider-s3:0.1.0")
implementation("com.revethq.buckets:revet-buckets-provider-gcs:0.1.0")
```

### Maven

```xml
<dependency>
    <groupId>com.revethq.buckets</groupId>
    <artifactId>revet-buckets-core</artifactId>
    <version>0.1.0</version>
</dependency>
```

## Core Concepts

### Bucket

Storage bucket configuration with encrypted credentials:

```kotlin
package com.revethq.buckets.domain

data class Bucket(
    val id: Long?,
    val uuid: UUID,
    val name: String,
    val provider: StorageProvider,
    val bucketName: String,
    val endpoint: String?,
    val region: String?,
    val accessKey: String,
    val secretKey: String,
    val presignedUrlDurationMinutes: Int,
    val isActive: Boolean,
    val removedAt: LocalDateTime?
)
```

Factory methods:
- `Bucket.create(...)` - New bucket with defaults (`isActive = true`, `presignedUrlDurationMinutes = 15`)
- `bucket.update(...)` - Immutable update with optional fields
- `bucket.deactivate()` - Soft delete (sets `isActive = false`, `removedAt = now`)

### Storage Providers

```kotlin
enum class StorageProvider {
    S3,
    GCS,
    AZURE_BLOB,
    MINIO
}
```

### Presigned URLs

```kotlin
data class PresignedUrl(
    val url: String,
    val expiresInMinutes: Int,
    val key: String
)
```

### File Metadata

```kotlin
data class FileMetadata(
    val size: Long,
    val contentType: String?,
    val lastModified: Instant
)
```

## Permission Model

Uses Revet IAM for authorization. See [permissions.md](./permissions.md) for full details.

**Actions:**

| Constant | Value |
|----------|-------|
| `Actions.Bucket.LIST` | `buckets:ListBuckets` |
| `Actions.Bucket.GET` | `buckets:GetBucket` |
| `Actions.Bucket.CREATE` | `buckets:CreateBucket` |
| `Actions.Bucket.UPDATE` | `buckets:UpdateBucket` |
| `Actions.Bucket.DELETE` | `buckets:DeleteBucket` |

**URN format:** `urn:revet:buckets:{tenantId}:bucket/{bucketUuid}`

## REST API

Base path: `/api/v1/buckets`

| Method | Endpoint | Action | Description |
|--------|----------|--------|-------------|
| GET | `/` | `ListBuckets` | List all buckets (query: `includeInactive`) |
| GET | `/{uuid}` | `GetBucket` | Get bucket by UUID |
| POST | `/` | `CreateBucket` | Create bucket configuration |
| PUT | `/{uuid}` | `UpdateBucket` | Update bucket configuration |
| DELETE | `/{uuid}` | `DeleteBucket` | Soft delete bucket |

## Related Documentation

- [permissions.md](./permissions.md) - URN builder, actions, IAM policy examples
- [storage.md](./storage.md) - Storage service interfaces, provider client SPI, encryption

## Extension Points

### StorageProviderClientFactory

Implement to add support for a new storage provider:

```kotlin
@ApplicationScoped
class CustomStorageProviderClientFactory : StorageProviderClientFactory {
    override fun supportedProviders(): Set<StorageProvider> = setOf(StorageProvider.AZURE_BLOB)

    override fun createClient(bucket: Bucket): StorageProviderClient {
        // Build provider-specific client using bucket credentials
    }
}
```

Provider factories are discovered via CDI `Instance<StorageProviderClientFactory>` injection. Add a new provider module with a `StorageProviderClientFactory` bean and it will be picked up automatically.

### EncryptionService

Implement to customize credential encryption:

```kotlin
@ApplicationScoped
class CustomEncryptionService : EncryptionService {
    override fun encrypt(plaintext: String): String { /* ... */ }
    override fun decrypt(ciphertext: String): String { /* ... */ }
}
```

Default: `AesEncryptionService` using AES-256-GCM with random IV, configured via `revet.encryption.key`.

## Configuration

| Property | Description | Default |
|----------|-------------|---------|
| `revet.encryption.key` | Base64-encoded 256-bit AES key for credential encryption | Required |
| `quarkus.http.port` | HTTP server port | `5052` |

## Key Constraints

- `name` and `bucketName` must not be blank
- `accessKey` and `secretKey` must not be blank
- `presignedUrlDurationMinutes` must be > 0
- MinIO provider requires a non-null `endpoint`
- Credentials (`accessKey`, `secretKey`) are encrypted at rest and never returned in API responses
- Delete is a soft delete (deactivation with timestamp)
