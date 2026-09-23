---
title: "Publisher Worker: MQTT Notifications"
description: "The publisher worker sends real-time MQTT notifications, resolving topics from each changed document's owner, shares and clients."
---

# Publisher

**Port:** `:4050`  
**Source:** `apps/workers/publisher/`

The publisher delivers real-time MQTT notifications to connected clients. On every MongoDB change event it resolves the set of MQTT topics that should be notified based on the document's `owner`, `shares`, and `clients` fields, then publishes a message to each topic via EMQX's HTTP API.

## Responsibilities

- Consume Kafka MongoDB change stream events (`MongoSourcePayload`).
- Derive MQTT topic paths from the document's access metadata.
- Bulk-publish QoS-2 messages to EMQX for each resolved topic.

## Topic Resolution

For a document with `owner`, `shares[]`, `groups[]` and `clients[]` in database `db` with `id` and
`collection`, `publish()` derives one topic per ownership attribute and per cross-scoped pair —
nine forms in all, listed once in [realtime.md → Topics and Schemas](../../api/realtime.md#topics-and-schemas). Duplicate topics
are deduplicated before publishing.

## Message Format

Each message is published with:

```json
{
  "topic": "<resolved-topic>",
  "qos": 2,
  "retain": false,
  "payload_encoding": "plain",
  "payload": "{\"id\":\"<entity_id>\",\"op\":\"c|u|d\",\"src\":\"<collection-type>\"}"
}
```

The `payload` contains only the entity ID, operation type, and source collection — subscribers fetch the full entity from the API if needed.

## Bulk Publishing

All resolved messages for a single change event are sent in one `publishMessageBulk` call to EMQX, minimizing HTTP round-trips.

## Infrastructure Dependencies

| Dependency | Usage |
|---|---|
| Kafka | Consumes MongoDB change stream events |
| EMQX | Receives bulk MQTT publish requests via HTTP API |

## Key Files

| File | Purpose |
|---|---|
| `app.service.ts` | `publish()` — topic resolution and bulk EMQX publish |
| `app.controller.ts` | `/status`, `/metrics` |

