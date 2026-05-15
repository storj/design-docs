tags: [ "satellite", "satellite/metainfo", "bucket eventing", "tidb" ]
---

# Bucket Eventing on TiDB

## Essentials

### Header

Date: 2026-04-29

Owner: [Kaloyan Raev](https://github.com/kaloyan-raev)

Accountable:
-

Consulted:
-

Informed:
- [#eng-object-storage-eventing](https://storj.slack.com/archives/C0939NQNFFG)

### Context

Phases 1 and 2 of bucket eventing are built on Google Cloud Spanner as the metainfo database. The core CDC mechanism is the Spanner Change Stream, a native database feature that records every relevant row mutation and makes it queryable via a streaming SQL API.

A variant of the Storj satellite uses TiDB as the metainfo database instead of Spanner. TiDB does have a CDC tool (TiCDC), but it is sink-based only — it outputs to Kafka or cloud storage and provides no API that application code can query directly, unlike Spanner's `READ_change_stream()` SQL function. Operating a Kafka cluster solely for bucket eventing would add significant infrastructure complexity with no benefit over a simpler approach.

This document describes how to implement bucket eventing for the TiDB metainfo database variant, preserving the same external guarantees and S3-compatible event format defined in Phases 1 and 2.

### Goals

- Implement bucket eventing for the TiDB metainfo database variant with the same external behavior as the Spanner-based implementation.
- Preserve at-least-once event delivery with no ordering or deduplication guarantees (same as Phase 1).
- Reuse the existing `bucket_eventing_configs` table, `eventing.Service`, S3 event transformation, and Pub/Sub publishing without modification.
- Keep Spanner-based bucket eventing fully operational and unaffected.

### Approach / Design

#### Why a Transactional Outbox

TiDB has a CDC tool (TiCDC) but it is sink-based only — it outputs to Kafka or cloud storage and has no API that application code can query directly. There are no transaction tags, no change stream queries, and no server-side notification hooks equivalent to Spanner's.

The most reliable approach is the **transactional outbox pattern**: when a metabase operation that should generate an event runs, the application writes an outbox row in the **same SQL transaction** as the object mutation. A separate worker polls and publishes these rows. Because both writes share one TiDB transaction, they are atomic — either both land or neither does, eliminating any window where an object is mutated but its event is lost.

This pattern also eliminates the Spanner-specific problem of inferring the S3 event type from a transaction tag after the fact. Because the event type is known at the call site in the metainfo layer, it is stored directly in the outbox row.

#### Outbox Table

The outbox table lives in the **TiDB metainfo database**, alongside the `objects` table. Writing the outbox row in the same SQL transaction as the object mutation gives atomicity without any special database features.

```sql
CREATE TABLE bucket_eventing_outbox (
    id               BIGINT        NOT NULL AUTO_INCREMENT,
    created_at       DATETIME(6)   NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    project_id       VARBINARY(16) NOT NULL,
    bucket_name      VARBINARY(1024) NOT NULL,
    object_key       BLOB          NOT NULL,
    version          BIGINT        NOT NULL,
    stream_id        VARBINARY(16),
    total_plain_size BIGINT,
    event_type       VARCHAR(64)   NOT NULL,
    PRIMARY KEY (id)
);
```

- `id`: Auto-incrementing primary key, used for ordered polling.
- `event_type`: Written directly by the metainfo layer. No inference from transaction tags needed.
- `stream_id` + `version`: Used to construct the S3-compatible `versionId` field (`StreamVersionID`), same as the Spanner path.

#### No Cursor Needed

The outbox acts as its own queue. Rows are deleted only after successful Pub/Sub delivery confirmation. Whatever remains in the table is by definition unprocessed, so on worker restart the worker simply reads from the beginning of the table and picks up where it left off. No separate cursor state is needed.

#### Writing to the Outbox

The outbox write is part of the **TiDB metabase transaction**, added by the TiDB adapter when `TransmitEvent: true` is set in the `TransactionOptions`. This is analogous to how the Spanner adapter sets `ExcludeTxnFromChangeStreams: false` on the Spanner transaction — it is a per-transaction option handled inside the adapter, invisible to the shared metabase logic.

The `TransmitEvent` flag is already threaded through all relevant metabase operations. On the TiDB adapter, `TransmitEvent: true` causes the transaction to additionally `INSERT` the outbox row alongside the object mutation, within the same SQL transaction.

The metainfo operations that trigger an outbox write (when `shouldTransmitEvent()` returns true) are the same as in Phases 1 and 2:

| Metainfo operation | Event type written to outbox |
|---|---|
| `CommitInlineObject` | `ObjectCreated:Put` |
| `CommitObjectWithSegments` | `ObjectCreated:Put` |
| `FinishCopyObject` | `ObjectCreated:Copy` |
| `FinishMoveObject` (destination) | `ObjectCreated:Copy` |
| `FinishMoveObject` (source) | `ObjectRemoved:Delete` |
| `DeleteObjectLastCommitted` (unversioned) | `ObjectRemoved:Delete` |
| `DeleteObjectLastCommitted` (versioned) | `ObjectRemoved:DeleteMarkerCreated` |
| `DeleteObjectExactVersion` | `ObjectRemoved:Delete` |
| `DeleteAllBucketObjects` | `ObjectRemoved:Delete` (one entry per deleted object) |

#### Outbox Worker

A new `TiDBEventingService` (or a TiDB-specific run mode of the existing `eventing.Service`) replaces the Spanner `changestream.Processor` loop. It runs as the same `satellite changestream` command, selected based on the configured metabase adapter.

The worker loop:

```
loop:
  rows = SELECT id, project_id, bucket_name, object_key, version,
                stream_id, total_plain_size, event_type
         FROM bucket_eventing_outbox
         ORDER BY id
         LIMIT 100

  if no rows:
    sleep $pollInterval  # e.g. 100ms
    continue

  for each row:
    look up bucket notification config (existing in-memory LRU cache + DB fallback)
    if no config or event type / filter does not match: skip
    translate row to EventRecord (see below)
    publish to Pub/Sub (existing publisher, PendingResult pattern)
    await confirmation
    DELETE FROM bucket_eventing_outbox WHERE id = $row.id
```

On startup, the worker reads from the beginning of the table. Rows that were not yet deleted (because the previous worker run did not confirm delivery) are reprocessed, fulfilling at-least-once delivery. No cursor state needs to be persisted separately.

#### Translating Outbox Rows to S3 Events

The outbox row contains all fields needed to build the `EventRecord` directly. The existing `notification.go` types (`Event`, `EventRecord`, `EncodeForS3Event`, `MatchEventType`, `MatchFilters`) are reused without modification.

| Outbox column | `EventRecord` field |
|---|---|
| `created_at` | `eventTime` |
| `event_type` | `eventName` (prefixed with `s3:`) |
| `bucket_name` | `s3.bucket.name`, `s3.bucket.arn` |
| `project_id` | Resolved to public project ID → `s3.bucket.ownerIdentity.principalId` |
| `object_key` | URL-encoded → `s3.object.key` |
| `total_plain_size` | `s3.object.size` |
| `stream_id` + `version` | `NewStreamVersionID(version, streamID)` hex-encoded → `s3.object.versionId` |
| `created_at` (Unix nanos) | 16-char uppercase hex → `s3.object.sequencer` |

The `ConvertModsToEvent()` function in `notification.go` is **not used** for the TiDB path — a new `ConvertOutboxRowToEvent()` function constructs the `EventRecord` directly from the typed outbox columns, with no JSON parsing or `spanner.NullJSON` involved.

#### No Changes to the Spanner Path

The `TransmitEvent` flag, `ExcludeTxnFromChangeStreams`, `changestream.Adapter`, `changestream.Processor`, `MetadataBatcher`, `partitionDrainer`, and `bucket_eventing_metadata` are all unchanged. The TiDB path is additive: new code in the metainfo layer and a new worker loop, selected at runtime based on the configured adapter.

The `notification.go` dependency on `spanner.NullJSON` and `cloud.google.com/go/spanner` is isolated to the Spanner path and does not need to be touched.

#### Relation to Phase 2 and Later Changes

The TiDB implementation inherits all current bucket eventing features:

- `bucket_eventing_configs` table (in satellite TiDB) is shared and unchanged.
- `PutBucketNotificationConfiguration` / `GetBucketNotificationConfiguration` S3 API endpoints work as designed in Phase 2.
- Bucket eventing is available to all users — project-level gating (`bucket-eventing.projects`) was removed after Phase 2.
- In-memory LRU caching for bucket notification configs is reused.
- `shouldTransmitEvent()` is reused without modification.
- Satellite-managed encryption requirement is enforced.
- Async Pub/Sub delivery via `PendingResult` is reused — the worker calls `Publish` and confirms delivery asynchronously before deleting the outbox row.

The only Spanner-specific mechanism that does not apply to TiDB is `ExcludeTxnFromChangeStreams` — TiDB has no equivalent, and cost filtering is handled instead by `shouldTransmitEvent()` already gating the outbox write.

## Disclaimers

### Anti-goals

- Using TiCDC with Kafka. TiCDC is available but sink-based only — it would require operating a Kafka cluster solely for bucket eventing, adding significant infrastructure complexity with no benefit over the outbox pattern.
- Exactly-once delivery. The same at-least-once guarantee from Phases 1 and 2 applies.

### Alternatives considered

**Outbox in the satellite TiDB database**

The outbox could be stored in the satellite TiDB database rather than the metainfo TiDB database.

Rejected because:
- The outbox write and the object mutation would be in different databases, so they cannot share an atomic transaction. A crash between the two writes loses the event with no recovery path.
- The atomicity guarantee of colocating the outbox in the metainfo database is strictly stronger and eliminates an entire class of failure.

**TiCDC → Kafka**

TiCDC can replicate all `objects` table changes to a Kafka topic. A consumer then filters and publishes S3 events.

Rejected because:
- Requires operating a Kafka cluster and a TiCDC cluster as permanent new infrastructure.
- TiCDC has no transaction tag equivalent, so distinguishing `ObjectCreated:Copy` from `ObjectCreated:Put` requires additional application-level state.
- The outbox pattern achieves the same result with no additional infrastructure.

## Reminders

### Security / Privacy

The private project ID must never appear in outbox rows written to logs, in event notification messages, or in error messages. The worker resolves the private project ID to the public project ID using the same `CachedPublicProjectIDs` wrapper as the Spanner path before constructing the `EventRecord`.

### Observability

The outbox worker should report:
- Lag: `created_at` of the oldest unprocessed row (`SELECT created_at FROM bucket_eventing_outbox ORDER BY id LIMIT 1`) vs. current time. A growing lag indicates the worker is falling behind. Equivalent to the "lagging watermark" metric from the Spanner path.
- Publish success / failure counts per bucket, reusing the existing `publish_success` and `publish_failed` eventkit events.

### Test plan

Non-exhaustive test plan:

- Verify that no outbox rows are written if no bucket has eventing enabled.
- Verify that an outbox row is written for each relevant metainfo operation when a bucket has eventing enabled.
- Verify that the event type stored in the outbox matches the operation (Put vs Copy vs Delete vs DeleteMarkerCreated).
- Verify that the worker correctly translates outbox rows to S3 event notifications and publishes to Pub/Sub.
- Verify at-least-once delivery: stop the worker, perform uploads and deletes, restart the worker, and confirm all events are delivered without loss.
- Verify that outbox entries are deleted only after successful Pub/Sub confirmation.
- Verify that removing a bucket notification config causes no further outbox writes for that bucket.
- Verify that the Spanner-based eventing path is unaffected.

### Rollout

The TiDB eventing path is selected automatically based on the configured metabase adapter. No new configuration flags are needed beyond those already defined in Phases 1 and 2. The `bucket_eventing_outbox` table is added to the TiDB metainfo database migration.

### Rollback

Disable the `satellite changestream` service for the TiDB metainfo database variant. Outbox rows accumulate in TiDB during downtime and are processed on restart.

## Out of scope

- Exactly-once delivery.
- TiCDC or Kafka-based CDC.
- Event types beyond those defined in Phases 1 and 2.
- Multiple destinations per bucket.
