# Storage Module

Package: `com.revethq.buckets.service`

## BucketService

```kotlin
package com.revethq.buckets.service

interface BucketService {
    fun getAllBuckets(includeInactive: Boolean = false): List<Bucket>
    fun getBucketById(id: Long): Bucket?
    fun getBucketByUuid(uuid: UUID): Bucket?
    fun createBucket(
        name: String,
        provider: StorageProvider,
        bucketName: String,
        accessKey: String,
        secretKey: String,
        endpoint: String? = null,
        region: String? = null,
        presignedUrlDurationMinutes: Int = 15
    ): Bucket
    fun updateBucket(id: Long, ...): Bucket?
    fun updateBucketByUuid(uuid: UUID, ...): Bucket?
    fun deleteBucket(id: Long): Boolean
    fun deleteBucketByUuid(uuid: UUID): Boolean
}
```

Implementation: `BucketServiceImpl` (in `web` module)

### Validation Rules

- `name` must not be blank
- `bucketName` must not be blank
- `accessKey` must not be blank
- `secretKey` must not be blank
- `presignedUrlDurationMinutes` must be > 0
- MinIO provider requires non-null `endpoint`
- Throws `IllegalArgumentException` on validation failure

## StorageService

```kotlin
package com.revethq.buckets.service

interface StorageService {
    fun generatePresignedUploadUrl(bucketId: Long, key: String, contentType: String?): PresignedUrl
    fun generatePresignedDownloadUrl(bucketId: Long, key: String): PresignedUrl
    fun checkFileExists(bucketId: Long, key: String): Boolean
    fun getFileMetadata(bucketId: Long, key: String): FileMetadata
    fun deleteFile(bucketId: Long, key: String): Boolean
}
```

Implementation: `StorageServiceImpl` (in `web` module)

Operations flow: fetch bucket config -> decrypt credentials -> create provider client -> delegate operation.

## EncryptionService

```kotlin
package com.revethq.buckets.service

interface EncryptionService {
    fun encrypt(plaintext: String): String
    fun decrypt(ciphertext: String): String
}
```

Implementation: `AesEncryptionService` (in `persistence-runtime` module)

- Algorithm: `AES/GCM/NoPadding`
- Key: 256-bit, Base64-decoded from `revet.encryption.key` config property
- IV: 12 bytes, randomly generated per encryption
- Auth tag: 128 bits
- Storage format: Base64(IV + ciphertext)

## BucketRepository

```kotlin
package com.revethq.buckets.repository

interface BucketRepository {
    fun findAll(includeInactive: Boolean = false): List<Bucket>
    fun findById(id: Long): Bucket?
    fun findByUuid(uuid: UUID): Bucket?
    fun save(bucket: Bucket): Bucket
    fun delete(id: Long): Boolean
}
```

Implementation: `BucketRepositoryImpl` (in `persistence-runtime` module)

- Encrypts `accessKey` and `secretKey` before persisting
- Decrypts credentials when reading from database
- Uses Hibernate Panache with `BucketEntity`

## Storage Provider SPI

### StorageProviderClient

```kotlin
package com.revethq.buckets.service.storage

interface StorageProviderClient {
    fun generatePresignedUploadUrl(key: String, contentType: String?): PresignedUrl
    fun generatePresignedDownloadUrl(key: String): PresignedUrl
    fun exists(key: String): Boolean
    fun getFileMetadata(key: String): FileMetadata
    fun delete(key: String): Boolean
}
```

### StorageProviderClientFactory

```kotlin
package com.revethq.buckets.service.storage

interface StorageProviderClientFactory {
    fun supportedProviders(): Set<StorageProvider>
    fun createClient(bucket: Bucket): StorageProviderClient
}
```

Each provider module implements this interface as a CDI bean. The `StorageClientFactoryImpl` discovers all factories via `Instance<StorageProviderClientFactory>` and delegates to the one matching the bucket's provider.

### StorageClientFactory

```kotlin
package com.revethq.buckets.service.storage

interface StorageClientFactory {
    fun createClient(bucket: Bucket): StorageProviderClient
}
```

Implementation: `StorageClientFactoryImpl` (in `web` module). Iterates over all `StorageProviderClientFactory` beans and selects the one whose `supportedProviders()` contains the bucket's provider.

## Provider Implementations

### S3 (`provider-s3`)

Package: `com.revethq.buckets.provider.s3`

- `S3StorageProviderClientFactory` - Supports `StorageProvider.S3` and `StorageProvider.MINIO`
- `S3StorageProviderClient` - Uses AWS SDK v2 (`S3Client`, `S3Presigner`)
- MinIO support via endpoint override on the S3 client builder
- Lazy-initializes SDK clients

### GCS (`provider-gcs`)

Package: `com.revethq.buckets.provider.gcs`

- `GcsStorageProviderClientFactory` - Supports `StorageProvider.GCS`
- `GcsStorageProviderClient` - Uses Google Cloud Storage library
- Supports region and custom endpoint configuration

### Azure Blob (`provider-azure-blob`)

Package: `com.revethq.buckets.provider.azure`

- `AzureBlobStorageProviderClientFactory` - Supports `StorageProvider.AZURE_BLOB`
- Currently a stub that throws `UnsupportedOperationException`

## Adding a New Provider

1. Create a new Gradle module (e.g., `provider-r2`)
2. Depend on `revet-buckets-core`
3. Implement `StorageProviderClientFactory` as a CDI `@ApplicationScoped` bean
4. Implement `StorageProviderClient` with provider-specific SDK calls
5. Add the module as a runtime dependency in the `web` module
6. The factory will be discovered automatically via CDI

## REST DTOs

### BucketDTO (Response)

```kotlin
data class BucketDTO(
    val id: Long?,
    val uuid: UUID,
    val name: String,
    val provider: StorageProvider,
    val bucketName: String,
    val endpoint: String?,
    val region: String?,
    val presignedUrlDurationMinutes: Int,
    val isActive: Boolean
)
```

Note: `accessKey` and `secretKey` are never included in responses.

### CreateBucketRequest

```kotlin
data class CreateBucketRequest(
    val name: String,
    val provider: StorageProvider,
    val bucketName: String,
    val accessKey: String,
    val secretKey: String,
    val endpoint: String? = null,
    val region: String? = null,
    val presignedUrlDurationMinutes: Int = 15
)
```

### UpdateBucketRequest

```kotlin
data class UpdateBucketRequest(
    val name: String? = null,
    val bucketName: String? = null,
    val endpoint: String? = null,
    val region: String? = null,
    val accessKey: String? = null,
    val secretKey: String? = null,
    val presignedUrlDurationMinutes: Int? = null,
    val isActive: Boolean? = null
)
```

## Persistence

Table: `revet_buckets`

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGINT | PK, auto-generated |
| `uuid` | UUID | Unique |
| `name` | VARCHAR | |
| `provider` | VARCHAR | Enum string |
| `bucket_name` | VARCHAR | |
| `endpoint` | VARCHAR | Nullable |
| `region` | VARCHAR | Nullable |
| `access_key` | TEXT | AES-256-GCM encrypted |
| `secret_key` | TEXT | AES-256-GCM encrypted |
| `presigned_url_duration_minutes` | INT | |
| `is_active` | BOOLEAN | |
| `removed_at` | TIMESTAMP | Nullable, for soft delete |
