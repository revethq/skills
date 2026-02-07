---
name: revet-core
description: Shared domain models for Revet libraries - Metadata, Identifier, SchemaValidation
---

# Revet Core Library

Lightweight shared domain models used across Revet libraries. Zero external dependencies beyond Kotlin stdlib and Java 8+ APIs.

## Dependency Coordinates

**Group ID:** `com.revethq`
**Artifact ID:** `revet-core`
**Version:** `0.1.0`

### Gradle

```kotlin
implementation("com.revethq:revet-core:0.1.0")
```

### Maven

```xml
<dependency>
    <groupId>com.revethq</groupId>
    <artifactId>revet-core</artifactId>
    <version>0.1.0</version>
</dependency>
```

## Domain Models

### Identifier

```kotlin
package com.revethq.core

data class Identifier(
    val system: String? = null,
    val value: String? = null
)
```

Represents an external identifier with a system namespace and value.

**Usage:**
```kotlin
val identifier = Identifier(
    system = "urn:oid:2.16.840.1.113883.4.1",
    value = "123-45-6789"
)
```

### SchemaValidation

```kotlin
package com.revethq.core

import java.time.OffsetDateTime
import java.util.UUID

data class SchemaValidation(
    val schemaId: UUID? = null,
    val isValid: Boolean = false,
    val validatedOn: OffsetDateTime? = null
)
```

Records schema validation state for a resource.

**Usage:**
```kotlin
val validation = SchemaValidation(
    schemaId = UUID.fromString("..."),
    isValid = true,
    validatedOn = OffsetDateTime.now()
)
```

### Metadata

```kotlin
package com.revethq.core

data class Metadata(
    val identifiers: List<Identifier> = emptyList(),
    val schemaValidations: List<SchemaValidation> = emptyList(),
    val properties: Map<String, Any> = emptyMap()
)
```

Aggregate metadata container combining:
- Multiple external identifiers
- Schema validation history
- Extensible key-value properties

**Usage:**
```kotlin
val metadata = Metadata(
    identifiers = listOf(
        Identifier(system = "saml", value = "user@idp.com"),
        Identifier(system = "scim", value = "external-id-123")
    ),
    schemaValidations = listOf(validation),
    properties = mapOf(
        "customField" to "value",
        "tier" to "premium"
    )
)
```

## Extension Points

### Properties Map

The `properties: Map<String, Any>` field in `Metadata` is the primary extension point:
- Add custom attributes without modifying core classes
- Store domain-specific metadata
- Maintain forward compatibility

### Multiple Identifier Systems

Support multiple external identifiers per resource:
- SAML NameID
- SCIM externalId
- OIDC subject
- Custom enterprise identifiers

### Multi-Schema Validation

Track validation against multiple schemas:
- Validate same resource against different schema versions
- Record validation history with timestamps

## Characteristics

- **Zero Dependencies**: Pure Kotlin + Java stdlib
- **Immutable by Default**: Data classes encourage immutable patterns
- **Type-Safe**: Full Kotlin null safety
- **Composition**: Metadata aggregates, doesn't extend

## Used By

- `revet-iam` - User, Group, Policy, and other entities include `metadata: Metadata`
- `revet-auth` - Application, Client, Scope entities use Metadata
