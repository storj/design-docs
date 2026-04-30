tags: [ "satellite", "satellite/metainfo", "bucket eventing", "tikv" ]
---

# Bucket Eventing on TiKV

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

A variant of the Storj satellite uses TiKV as the metainfo database instead of Spanner. TiKV is a distributed key-value store with no native change stream, no CDC API queryable by application code, no transaction tags, and no server-side triggers. TiCDC (the CDC tool in the TiKV ecosystem) requires TiDB on top of TiKV and outputs only to external sinks like Kafka — it is not available in a raw TiKV deployment.

This document describes how to implement bucket eventing for the TiKV metainfo database variant, preserving the same external guarantees and S3-compatible event format defined in Phases 1 and 2.

### Goals

- Implement bucket eventing for the TiKV metainfo database variant with the same external behavior as the Spanner-based implementation.
- Preserve at-least-once event delivery with no ordering or deduplication guarantees (same as Phase 1).
- Reuse the existing `bucket_eventing_configs` table, `eventing.Service`, S3 event transformation, and Pub/Sub publishing without modification.
- Keep Spanner-based bucket eventing fully operational and unaffected.

### Approach / Design

#### Why a Transactional Outbox

TiKV provides no CDC mechanism that application code can consume. TiCDC is not available without TiDB. RawKV CDC is an experimental feature designed for cluster-to-cluster replication, not fine-grained application event delivery. There are no transaction tags, no change stream queries, and no server-side notification hooks.

The only reliable approach is the **transactional outbox pattern**: when a metabase operation that should generate an event runs, the application writes an outbox entry in the **same TiKV transaction** as the object mutation. A separate worker reads and publishes these entries. Because both writes are part of the same TiKV transaction, they are atomic — either both land or neither does, eliminating any window where an object is mutated but its event is lost.

This pattern also eliminates the Spanner-specific problem of inferring the S3 event type from a transaction tag after the fact. Because the event type is known at the call site in the metainfo layer, it is stored directly in the outbox entry.

#### Outbox Key Design

The outbox entries live in TiKV in a dedicated key namespace, written in the same transaction as the object mutation. TiKV's transactional API (TxnKV) supports arbitrary key-value pairs within a single transaction, so the outbox entry and the object row are committed atomically.

Keys are structured to support efficient ordered range scans by the worker:

```
outbox/{big-endian commit timestamp (8 bytes)}/{stream_id (16 bytes)}
```

- **Commit timestamp**: TiKV assigns a commit timestamp (TSO) to every transaction. Using it as the key prefix gives natural chronological ordering. The worker scans the full `outbox/` namespace from the beginning on each iteration, processing whatever entries remain.
- **`stream_id` suffix**: Makes the key unique when a single transaction writes multiple outbox entries (e.g. `FinishMoveObject` produces both a delete event for the source and a copy event for the destination).

The value is a serialized struct (e.g. protobuf or msgpack) containing:

```
project_id       bytes(16)
bucket_name      bytes
object_key       bytes
version          int64
stream_id        bytes(16)
total_plain_size int64
event_type       string   // e.g. "ObjectCreated:Put"
```

- `event_type`: Written directly by the metainfo layer. No inference from transaction tags needed.
- `stream_id` + `version`: Used to construct the S3-compatible `versionId` field (`StreamVersionID`), same as the Spanner path.

#### No Cursor Needed

The outbox acts as its own queue. Entries are deleted from TiKV only after successful Pub/Sub delivery confirmation. Whatever remains in the `outbox/` namespace is by definition unprocessed, so on worker restart the worker simply scans from the beginning of the namespace and picks up where it left off. No separate cursor or watermark state is needed.

#### Writing to the Outbox

The outbox write is part of the **TiKV metabase transaction**, added by the TiKV adapter when `TransmitEvent: true` is set in the `TransactionOptions`. This is analogous to how the Spanner adapter sets `ExcludeTxnFromChangeStreams: false` on the Spanner transaction — it is a per-transaction option handled inside the adapter, invisible to the shared metabase logic.

The `TransmitEvent` flag is already threaded through all relevant metabase operations. On the TiKV adapter, `TransmitEvent: true` causes the transaction to additionally write the outbox key-value pair alongside the object mutation, within the same atomic commit.

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

A new `TiKVEventingService` (or a TiKV-specific run mode of the existing `eventing.Service`) replaces the Spanner `changestream.Processor` loop. It runs as the same `satellite changestream` command, selected based on the configured metabase adapter.

The worker loop:

```
loop:
  entries = TiKV Scan(
    start = outbox/{0},
    end   = outbox/{max},
    limit = 100
  )

  if no entries:
    sleep $pollInterval  # e.g. 100ms
    continue

  for each entry:
    decode value → project_id, bucket_name, object_key, version,
                   stream_id, total_plain_size, event_type
    look up bucket notification config (existing in-memory LRU cache + DB fallback)
    if no config or event type / filter does not match: skip
    translate entry to EventRecord (see below)
    publish to Pub/Sub (existing publisher, PendingResult pattern)
    await confirmation
    delete entry from TiKV
```

On startup, the worker scans from the beginning of the `outbox/` namespace. Entries that were not yet deleted (because the previous worker run did not confirm delivery) are reprocessed, fulfilling at-least-once delivery. No cursor state needs to be persisted separately.

#### Translating Outbox Entries to S3 Events

The outbox entry contains all fields needed to build the `EventRecord` directly. The existing `notification.go` types (`Event`, `EventRecord`, `EncodeForS3Event`, `MatchEventType`, `MatchFilters`) are reused without modification.

| Outbox field | `EventRecord` field |
|---|---|
| commit timestamp (from key prefix) | `eventTime` |
| `event_type` | `eventName` (prefixed with `s3:`) |
| `bucket_name` | `s3.bucket.name`, `s3.bucket.arn` |
| `project_id` | Resolved to public project ID → `s3.bucket.ownerIdentity.principalId` |
| `object_key` | URL-encoded → `s3.object.key` |
| `total_plain_size` | `s3.object.size` |
| `stream_id` + `version` | `NewStreamVersionID(version, streamID)` hex-encoded → `s3.object.versionId` |
| commit timestamp (Unix nanos) | 16-char uppercase hex → `s3.object.sequencer` |

The `ConvertModsToEvent()` function in `notification.go` is **not used** for the TiKV path — a new `ConvertOutboxRowToEvent()` function constructs the `EventRecord` directly from the typed outbox fields, with no JSON parsing or `spanner.NullJSON` involved.

#### No Changes to the Spanner Path

The `TransmitEvent` flag, `ExcludeTxnFromChangeStreams`, `changestream.Adapter`, `changestream.Processor`, `MetadataBatcher`, `partitionDrainer`, and `bucket_eventing_metadata` are all unchanged. The TiKV path is additive: new code in the metainfo layer and a new worker loop, selected at runtime based on the configured adapter.

The `notification.go` dependency on `spanner.NullJSON` and `cloud.google.com/go/spanner` is isolated to the Spanner path and does not need to be touched.

#### Relation to Phase 2 and Later Changes

The TiKV implementation inherits all current bucket eventing features:

- `bucket_eventing_configs` table (in satellite TiDB) is shared and unchanged.
- `PutBucketNotificationConfiguration` / `GetBucketNotificationConfiguration` S3 API endpoints work as designed in Phase 2.
- Bucket eventing is available to all users — project-level gating (`bucket-eventing.projects`) was removed after Phase 2.
- In-memory LRU caching for bucket notification configs is reused.
- `shouldTransmitEvent()` is reused without modification.
- Satellite-managed encryption requirement is enforced.
- Async Pub/Sub delivery via `PendingResult` is reused — the worker calls `Publish` and confirms delivery asynchronously before deleting the outbox entry.

The only Spanner-specific mechanism that does not apply to TiKV is `ExcludeTxnFromChangeStreams` — TiKV has no equivalent, and cost filtering is handled instead by `shouldTransmitEvent()` already gating the outbox write.

## Disclaimers

### Anti-goals

- Polling the `objects` table in TiKV directly for changed keys as a CDC mechanism. This would require deep TiKV internals knowledge, is fragile, and provides no advantage over the outbox pattern.
- Using TiCDC or Kafka. The metainfo database is raw TiKV; TiDB is only the satellite database. TiCDC requires TiDB on top of TiKV and is not available here.
- Exactly-once delivery. The same at-least-once guarantee from Phases 1 and 2 applies.

### Alternatives considered

**Outbox in TiDB (satellite database)**

The outbox could be stored as a SQL table in the satellite TiDB database, written after the TiKV object mutation completes. TiDB provides AUTO_INCREMENT cursors, standard SQL polling (`WHERE id > $last_id ORDER BY id`), TTL, and familiar tooling.

Rejected because:
- The outbox write and the object mutation are in different databases, so they cannot share an atomic transaction. A crash between the two writes loses the event with no recovery path.
- The satellite TiDB may not yet exist at the time of TiKV metainfo migration; the TiKV metabase should not depend on it for correctness.
- The atomicity guarantee of the TiKV-native outbox is strictly stronger and eliminates an entire class of failure.

**Polling the `objects` table in TiKV directly**

Query TiKV for recently modified objects by timestamp, using TiKV's MVCC read capabilities. This avoids a separate outbox table.

Rejected because:
- TiKV MVCC timestamps are internal and not reliably accessible to application code via the client-go API.
- Requires scanning large key ranges to find recently changed objects, which is expensive and does not scale.
- Cannot distinguish between the different operation types (Put vs Copy vs Delete) needed to determine the S3 event type.
- Provides no advantage over the outbox pattern.

**TiKV RawKV CDC**

An experimental feature available in TiKV ≥ 6.2 with API V2 enabled. Requires deploying a separate TiKV-CDC component and is designed for cluster-to-cluster replication.

Rejected because:
- Experimental and not production-ready for fine-grained event delivery.
- Requires TiKV API V2, which may not match the existing deployment configuration.
- Outputs to another TiKV cluster, not to application code directly.
- Adds significant operational complexity (new TiKV-CDC cluster to operate).

## Reminders

### Security / Privacy

The private project ID must never appear in outbox rows written to logs, in event notification messages, or in error messages. The worker resolves the private project ID to the public project ID using the same `CachedPublicProjectIDs` wrapper as the Spanner path before constructing the `EventRecord`.

### Observability

The outbox worker should report:
- Lag: commit timestamp of the oldest entry in the `outbox/` namespace (a single `Scan` with `limit=1`) vs. current time. A growing lag indicates the worker is falling behind. Equivalent to the "lagging watermark" metric from the Spanner path.
- Publish success / failure counts per bucket, reusing the existing `publish_success` and `publish_failed` eventkit events.

### Test plan

Non-exhaustive test plan:

- Verify that no outbox entries are written if no bucket has eventing enabled.
- Verify that an outbox entry is written for each relevant metainfo operation when a bucket has eventing enabled.
- Verify that the event type stored in the outbox matches the operation (Put vs Copy vs Delete vs DeleteMarkerCreated).
- Verify that the worker correctly translates outbox rows to S3 event notifications and publishes to Pub/Sub.
- Verify at-least-once delivery: stop the worker, perform uploads and deletes, restart the worker, and confirm all events are delivered without loss.
- Verify that outbox entries are deleted only after successful Pub/Sub confirmation.
- Verify that removing a bucket notification config causes no further outbox writes for that bucket.
- Verify that the Spanner-based eventing path is unaffected.

### Rollout

The TiKV eventing path is selected automatically based on the configured metabase adapter. No new configuration flags are needed beyond those already defined in Phases 1 and 2. The outbox key namespace is created on first use in TiKV — no schema migration is required.

### Rollback

Disable the `satellite changestream` service for the TiKV metainfo database variant. Outbox entries accumulate in TiKV during downtime and are processed on restart.

## Out of scope

- Exactly-once delivery.
- TiCDC or Kafka-based CDC for TiKV.
- Event types beyond those defined in Phases 1 and 2.
- Multiple destinations per bucket.
