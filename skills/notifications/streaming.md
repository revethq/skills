# Streaming Module

Package: `com.revethq.notifications.web`

## Notification Streaming

Consumers pull notifications for a subscription via two modes: cursor-based streaming and date-range queries.

**Endpoint:** `GET /subscriptions/{id}/notifications`

Both modes filter notifications against the subscription's `resourcePatterns` and `eventTypes`.

## Cursor-Based Streaming

Use a `token` query parameter to consume notifications sequentially.

### Request

```
GET /subscriptions/{id}/notifications?token={token}&limit=50
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `token` | String | (none) | Base64-encoded cursor; omit for first call |
| `limit` | Int | 50 | Max notifications to return |

### StreamToken

```kotlin
data class StreamToken(val lastSequenceId: Long) {
    fun encode(): String      // Base64 JSON: {"s": lastSequenceId}

    companion object {
        fun decode(token: String): StreamToken
    }
}
```

### StreamResponse

```kotlin
data class StreamResponse(
    val notifications: List<NotificationResponse>,
    val nextToken: String?,
    val hasMore: Boolean,
)
```

### Usage Pattern

```
# First call — no token, starts from beginning
GET /subscriptions/{id}/notifications?limit=100

# Response includes nextToken
{
    "notifications": [...],
    "nextToken": "eyJzIjogNDJ9",
    "hasMore": true
}

# Subsequent calls — pass nextToken
GET /subscriptions/{id}/notifications?token=eyJzIjogNDJ9&limit=100

# Continue until hasMore is false
```

### How It Works

1. Decode `StreamToken` to get `lastSequenceId`
2. Query notifications with `sequenceId > lastSequenceId`
3. Filter by subscription's `resourcePatterns` and `eventTypes`
4. Return matching notifications with a new `nextToken` for the next call
5. `hasMore = true` if more notifications may exist beyond this batch

## Date-Range Queries

Query notifications within a time window with pagination.

### Request

```
GET /subscriptions/{id}/notifications?from={from}&to={to}&page=0&size=50
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `from` | OffsetDateTime | (required) | Start of range |
| `to` | OffsetDateTime | (required) | End of range |
| `page` | Int | 0 | Page number |
| `size` | Int | 50 | Page size |

### DateRangeResponse

```kotlin
data class DateRangeResponse(
    val notifications: List<NotificationResponse>,
    val page: Int,
    val size: Int,
    val hasMore: Boolean,
)
```

### Usage Pattern

```
# Query notifications from a time range
GET /subscriptions/{id}/notifications?from=2026-03-01T00:00:00Z&to=2026-03-07T00:00:00Z&page=0&size=100
```

## Mode Selection

The endpoint selects the response mode based on query parameters:

- If `token` is present → cursor-based `StreamResponse`
- If `from`/`to` are present → paginated `DateRangeResponse`
- If neither → cursor-based from the beginning
