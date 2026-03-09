# Delivery Module

## Channel SPI

Package: `com.revethq.notifications.spi`

### Channel Interface

Implement to add a new delivery channel:

```kotlin
interface Channel {
    val channelType: String

    fun deliver(request: DeliveryRequest): DeliveryResult

    fun validateConfig(config: Map<String, Any>): List<String>
}
```

### DeliveryRequest

```kotlin
data class DeliveryRequest(
    val deliveryId: UUID,
    val notificationId: UUID,
    val subscriptionId: UUID,
    val resourceUrn: String,
    val eventType: String,
    val subject: String,
    val body: String? = null,
    val data: Map<String, Any> = emptyMap(),
    val channelConfig: Map<String, Any> = emptyMap(),
)
```

### DeliveryResult

```kotlin
data class DeliveryResult(
    val success: Boolean,
    val error: String? = null,
) {
    companion object {
        fun delivered(): DeliveryResult
        fun failed(error: String): DeliveryResult
    }
}
```

## Adding a Channel

1. Implement `Channel` as a CDI bean:

```kotlin
@ApplicationScoped
class WebhookChannel : Channel {
    override val channelType: String = "webhook"

    override fun deliver(request: DeliveryRequest): DeliveryResult {
        val url = request.channelConfig["url"] as? String
            ?: return DeliveryResult.failed("Missing webhook URL")
        // POST notification to webhook URL
        return DeliveryResult.delivered()
    }

    override fun validateConfig(config: Map<String, Any>): List<String> {
        val errors = mutableListOf<String>()
        if (config["url"] !is String) errors.add("url is required")
        return errors
    }
}
```

2. Channels are discovered via CDI `Instance<Channel>` injection — no registration needed.

3. Create subscriptions that reference the channel:

```kotlin
// POST /subscriptions
{
    "name": "webhook-orders",
    "resourcePatterns": ["urn:*:orders:*:order/**"],
    "eventTypes": ["order.created.*"],
    "channelType": "webhook",
    "channelConfig": { "url": "https://example.com/hooks/orders" }
}
```

## Built-in: ConsoleChannel

Writes notifications to stdout. Useful for development and testing.

- `channelType = "console"`
- Always validates successfully
- Always returns `DeliveryResult.delivered()`

## Retry Scheduler

Package: `com.revethq.notifications.scheduler`

The `RetryScheduler` runs as a Quarkus scheduled task:

- **Interval:** every 1 second
- **Batch size:** 50 records per cycle
- **Max attempts:** 7

### Retry Backoff Schedule

| After Attempt | Wait Before Next Retry |
|---------------|----------------------|
| 1 | 30 seconds |
| 2 | 2 minutes |
| 3 | 8 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |
| 6+ | 8 hours |

### Processing Flow

1. Fetch PENDING and FAILED records where `nextRetryOn <= now` (batch of 50)
2. Look up matching `Channel` by `channelType`
3. Build `DeliveryRequest` from notification + subscription data
4. Call `channel.deliver(request)`
5. On success: set status to DELIVERED, record `deliveredOn`
6. On failure: increment `attemptCount`, set `nextRetryOn` per backoff schedule
7. If `attemptCount >= 7`: set status to ABANDONED
8. If no `Channel` found for `channelType`: set status to ABANDONED

### Manual Retry

```
POST /deliveries/{id}/retry
```

Resets a FAILED or ABANDONED delivery record to PENDING for re-processing. Returns 400 if the record is already DELIVERED.
