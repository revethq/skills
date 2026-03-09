---
name: revet-notifications
description: Event notification system - publish, subscribe, multi-channel delivery with retry, and cursor-based streaming
---

# Revet Notifications Library

Event notification system for Kotlin/Quarkus applications. Publish notifications to URN-based resources, subscribe with pattern matching, deliver via pluggable channels with automatic retry, and consume via cursor-based streaming or date-range queries.

## Dependency Coordinates

**Group ID:** `com.revethq.notifications`
**Version:** `0.1.0`

### Modules

| Artifact | Purpose |
|----------|---------|
| `core` | Domain models, SPI interfaces, URN/event pattern matching (no Quarkus dependency) |
| `web` | JAX-RS REST API, DTOs, exception mappers |
| `persistence-runtime` | Hibernate Panache persistence, service implementations |
| `scheduler` | Asynchronous delivery processing with exponential backoff retry |

### Gradle

```kotlin
implementation("com.revethq.notifications:core:0.1.0")
implementation("com.revethq.notifications:web:0.1.0")
implementation("com.revethq.notifications:persistence-runtime:0.1.0")
implementation("com.revethq.notifications:scheduler:0.1.0")
```

## Core Concepts

### Notification

An event record published to a URN-addressed resource:

```kotlin
package com.revethq.notifications.domain

data class Notification(
    var id: UUID,
    var sequenceId: Long? = null,
    var resourceUrn: String,
    var eventType: String,
    var subject: String,
    var body: String? = null,
    var source: String? = null,
    var data: Map<String, Any> = emptyMap(),
    var metadata: Metadata = Metadata(),
    var occurredOn: OffsetDateTime? = null,
    var createdOn: OffsetDateTime? = null,
)
```

- `sequenceId` is auto-generated on insert (monotonically increasing)
- `resourceUrn` addresses the resource this event relates to
- `eventType` categorizes the event (e.g., `user.created.v1`)
- `data` carries arbitrary event payload
- `metadata` uses `com.revethq.core.Metadata`

### Subscription

Declares interest in notifications matching URN and event type patterns:

```kotlin
data class Subscription(
    var id: UUID,
    var name: String,
    var description: String? = null,
    var tenantId: String? = null,
    var resourcePatterns: List<String>,
    var eventTypes: List<String> = emptyList(),
    var channelType: String? = null,
    var channelConfig: Map<String, Any>? = null,
    var active: Boolean = true,
    var metadata: Metadata = Metadata(),
    var createdOn: OffsetDateTime? = null,
    var updatedOn: OffsetDateTime? = null,
)
```

- `resourcePatterns` — URN patterns with `*` / `**` wildcards
- `eventTypes` — event type patterns with `*` / `**` wildcards
- `channelType` — identifies which `Channel` delivers notifications
- `channelConfig` — channel-specific configuration (e.g., webhook URL, email address)
- Unique constraint on `(name, tenant_id)`

### DeliveryRecord

Tracks delivery attempts for a notification-subscription pair:

```kotlin
data class DeliveryRecord(
    var id: UUID,
    var notificationId: UUID,
    var subscriptionId: UUID,
    var channelType: String,
    var status: DeliveryStatus,
    var attemptCount: Int = 0,
    var lastAttemptedOn: OffsetDateTime? = null,
    var deliveredOn: OffsetDateTime? = null,
    var nextRetryOn: OffsetDateTime? = null,
    var lastError: String? = null,
    var createdOn: OffsetDateTime? = null,
)
```

### DeliveryStatus

```kotlin
enum class DeliveryStatus {
    PENDING,     // Awaiting delivery attempt
    DELIVERED,   // Successfully delivered
    FAILED,      // Attempt failed, will retry
    ABANDONED,   // Max attempts exceeded or no channel available
}
```

## Pattern Matching

### URN Patterns

URN format: `urn:{namespace}:{service}:{tenant}:{resourceType}/{resourceId}`

```kotlin
object Urn {
    fun parse(urn: String): Urn?
    fun parseOrThrow(urn: String): Urn
    fun matchesPattern(urn: String, pattern: String): Boolean
}
```

- `*` matches a single segment
- `**` matches any sequence of segments

Examples:
```kotlin
Urn.matchesPattern("urn:revet:users:acme:user/123", "urn:*:users:*:user/**") // true
Urn.matchesPattern("urn:revet:users:acme:user/org/team/456", "urn:*:users:*:user/**") // true
```

### Event Type Patterns

Dot-separated event types with wildcard matching:

```kotlin
object EventType {
    fun matches(eventType: String, pattern: String): Boolean
}
```

- `*` matches a single segment
- `**` matches multiple segments

Examples:
```kotlin
EventType.matches("user.created.v1", "user.created.*")  // true
EventType.matches("user.identity.created", "user.**.created") // true
```

## REST API

### Notifications — `/notifications`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/notifications` | Publish a notification |
| GET | `/notifications/{id}` | Get notification by ID |
| GET | `/notifications/{id}/deliveries` | Get delivery records for notification |

### Subscriptions — `/subscriptions`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/subscriptions` | Create subscription |
| GET | `/subscriptions/{id}` | Get subscription by ID |
| GET | `/subscriptions` | List subscriptions (query: `tenantId`, `page`, `size`) |
| PUT | `/subscriptions/{id}` | Update subscription |
| DELETE | `/subscriptions/{id}` | Delete subscription |
| PATCH | `/subscriptions/{id}/activate` | Enable subscription |
| PATCH | `/subscriptions/{id}/deactivate` | Disable subscription |

### Deliveries

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/subscriptions/{id}/deliveries` | Get deliveries for subscription (query: `status`, `page`, `size`) |
| POST | `/deliveries/{id}/retry` | Manually retry a failed delivery |

### Notification Streaming — `/subscriptions/{id}/notifications`

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/subscriptions/{id}/notifications?token=...&limit=50` | Cursor-based pull |
| GET | `/subscriptions/{id}/notifications?from=...&to=...&page=0&size=50` | Date-range query |

See [streaming.md](./streaming.md) for cursor and date-range details.

## Related Documentation

- [delivery.md](./delivery.md) - Channel SPI, retry scheduler, delivery workflow
- [streaming.md](./streaming.md) - Cursor-based streaming and date-range queries

## Request/Response DTOs

### PublishNotificationRequest

```kotlin
data class PublishNotificationRequest(
    val resourceUrn: String,
    val eventType: String,
    val subject: String,
    val body: String? = null,
    val source: String? = null,
    val data: Map<String, Any> = emptyMap(),
    val occurredOn: OffsetDateTime? = null,
)
```

### NotificationResponse

```kotlin
data class NotificationResponse(
    val id: UUID,
    val sequenceId: Long?,
    val resourceUrn: String,
    val eventType: String,
    val subject: String,
    val body: String?,
    val source: String?,
    val data: Map<String, Any>,
    val occurredOn: OffsetDateTime?,
    val createdOn: OffsetDateTime?,
)
```

### CreateSubscriptionRequest

```kotlin
data class CreateSubscriptionRequest(
    val name: String,
    val description: String? = null,
    val tenantId: String? = null,
    val resourcePatterns: List<String>,
    val eventTypes: List<String> = emptyList(),
    val channelType: String? = null,
    val channelConfig: Map<String, Any>? = null,
)
```

### UpdateSubscriptionRequest

```kotlin
data class UpdateSubscriptionRequest(
    val name: String,
    val description: String? = null,
    val resourcePatterns: List<String>,
    val eventTypes: List<String> = emptyList(),
    val channelType: String? = null,
    val channelConfig: Map<String, Any>? = null,
)
```

### SubscriptionResponse

```kotlin
data class SubscriptionResponse(
    val id: UUID,
    val name: String,
    val description: String?,
    val tenantId: String?,
    val resourcePatterns: List<String>,
    val eventTypes: List<String>,
    val channelType: String?,
    val channelConfig: Map<String, Any>?,
    val active: Boolean,
    val createdOn: OffsetDateTime?,
    val updatedOn: OffsetDateTime?,
)
```

### DeliveryResponse

```kotlin
data class DeliveryResponse(
    val id: UUID,
    val notificationId: UUID,
    val subscriptionId: UUID,
    val channelType: String,
    val status: DeliveryStatus,
    val attemptCount: Int,
    val lastAttemptedOn: OffsetDateTime?,
    val deliveredOn: OffsetDateTime?,
    val nextRetryOn: OffsetDateTime?,
    val lastError: String?,
    val createdOn: OffsetDateTime?,
)
```

### PageResponse

```kotlin
data class PageResponse<T>(
    val content: List<T>,
    val page: Int,
    val size: Int,
    val hasMore: Boolean,
)
```

## Exception Handling

| Exception | HTTP Status | When |
|-----------|-------------|------|
| `NotificationNotFoundException` | 404 | Notification ID not found |
| `SubscriptionNotFoundException` | 404 | Subscription ID not found |
| `SubscriptionConflictException` | 409 | Duplicate `(name, tenantId)` |
| `DeliveryNotRetryableException` | 400 | Retry on already-delivered record |

All return JSON: `{"error": "message"}`

## Service Interfaces

```kotlin
interface NotificationService {
    fun publish(notification: Notification): Notification
    fun findById(id: UUID): Notification?
    fun findBySequenceIdGreaterThan(sequenceId: Long, limit: Int): List<Notification>
    fun findByCreatedOnBetween(from: OffsetDateTime, to: OffsetDateTime, page: Int, size: Int): List<Notification>
}

interface SubscriptionService {
    fun create(subscription: Subscription): Subscription
    fun findById(id: UUID): Subscription?
    fun list(tenantId: String?, page: Int, size: Int): List<Subscription>
    fun update(id: UUID, subscription: Subscription): Subscription
    fun delete(id: UUID): Boolean
    fun activate(id: UUID): Subscription
    fun deactivate(id: UUID): Subscription
}

interface DeliveryService {
    fun findByNotificationId(notificationId: UUID): List<DeliveryRecord>
    fun findBySubscriptionId(subscriptionId: UUID, status: String?, page: Int, size: Int): List<DeliveryRecord>
    fun findById(id: UUID): DeliveryRecord?
    fun retry(id: UUID): DeliveryRecord
}
```

## Configuration

| Property | Description | Default |
|----------|-------------|---------|
| `quarkus.datasource.db-kind` | Database type | `postgresql` |
| `quarkus.datasource.jdbc.url` | JDBC connection URL | `jdbc:postgresql://localhost:5432/revet_notifications` |
| `quarkus.datasource.username` | Database username | `revet` |
| `quarkus.datasource.password` | Database password | `revet` |

## Database Tables

| Table | Purpose |
|-------|---------|
| `revet_notifications` | Notification records (JSON columns: `data`, `metadata`) |
| `revet_subscriptions` | Subscription records (JSON columns: `resource_patterns`, `event_types`, `channel_config`, `metadata`). Unique constraint: `(name, tenant_id)` |
| `revet_delivery_records` | Delivery tracking (FK to notifications and subscriptions) |

## Key Workflow

1. Client publishes via `POST /notifications`
2. System stores notification, generates `sequenceId`
3. `NotificationMatcher` finds active subscriptions matching `resourcePatterns` and `eventTypes`
4. Creates PENDING `DeliveryRecord` for each matching subscription with a `channelType`
5. `RetryScheduler` picks up PENDING records every 1 second
6. Invokes matching `Channel.deliver()`
7. Updates record: DELIVERED on success, FAILED with backoff on failure, ABANDONED after max attempts
