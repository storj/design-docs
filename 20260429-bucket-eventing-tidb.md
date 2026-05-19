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
- Reuse the existing `bucket_eventing_configs` table, S3 event transformation, and Pub/Sub publishing without modification.
- Refactor `eventing.Service` to be backend-agnostic via an `EventSource` interface, so the Spanner and TiDB paths share all filtering, translation, and publishing logic.
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
    id               BIGINT          NOT NULL AUTO_INCREMENT,
    created_at       DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    project_id       VARBINARY(16)   NOT NULL,
    bucket_name      VARBINARY(64)   NOT NULL,
    object_key       VARBINARY(4000) NOT NULL,
    version          BIGINT          NOT NULL,
    stream_id        VARBINARY(16),
    total_plain_size BIGINT,
    event_type       VARCHAR(64)     NOT NULL,
    PRIMARY KEY (id)
);
```

- `id`: Auto-incrementing primary key, used for ordered polling.
- `event_type`: Written directly by the metainfo layer. No inference from transaction tags needed.
- `stream_id` + `version`: Used to construct the S3-compatible `versionId` field (`StreamVersionID`), same as the Spanner path.

#### No Persistent Cursor Needed

The outbox acts as its own queue. Rows are deleted only after successful Pub/Sub delivery confirmation. Whatever remains in the table is by definition unprocessed, so on worker restart the worker simply reads from the beginning of the table and picks up where it left off. No cursor state needs to be persisted to a database.

The worker does maintain an in-memory `lastSeenID` while running, so the reader goroutine can skip rows already handed to the publisher goroutine within the same run. This is purely a performance optimization — on restart it resets to 0, causing any undeleted rows to be re-read and re-published, which is correct under at-least-once delivery.

#### Writing to the Outbox

The outbox write is part of the **TiDB metabase transaction**, added by the TiDB adapter when `TransmitEvent: true` is set in the `TransactionOptions`. This is analogous to how the Spanner adapter sets `ExcludeTxnFromChangeStreams` to `!TransmitEvent` on the Spanner transaction — it is a per-transaction option handled inside the adapter, invisible to the shared metabase logic.

The `TransmitEvent` flag is already threaded through all relevant metabase operations and present in the WIP TiDB adapter, but the outbox INSERT is not yet implemented. There are two integration points:

- **Commit / copy / move**: `TiDBAdapter.WithTx` (in `commit_object.go`) creates a `tidbTransactionAdapter`. The `TransmitEvent` flag needs to be propagated into the adapter so that `finalizeObjectCommit`, `commitPendingCopyObject`, and `objectMove` can include the outbox INSERT in the same transaction.
- **Delete operations**: `TiDBAdapter.deleteObjectExactVersion`, `deleteObjectLastCommittedPlain`, `DeleteObjectLastCommittedVersioned`, etc. open their own `txutil.WithTx` transactions directly, with no `tidbTransactionAdapter` involved. The outbox INSERT needs to be added inside each of these transaction closures when `TransmitEvent: true`.

On the TiDB adapter, `TransmitEvent: true` causes the transaction to additionally `INSERT` the outbox row alongside the object mutation, within the same SQL transaction.

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

#### EventSource Interface

`eventing.Service` is refactored to be backend-agnostic by introducing an `EventSource` interface. The service holds an `EventSource` instead of a `changestream.Adapter`, and `Run()` drives it. All filtering, project-ID resolution, translation to S3 `EventRecord`, and publishing logic stays in the service and is shared by both backends.

```go
// EventSource abstracts over the backend-specific record delivery loop.
// Implementations decode backend records into ChangeEvents and call fn for each one.
// Run blocks until ctx is cancelled or a permanent error occurs.
type EventSource interface {
    Run(ctx context.Context, fn func(event ChangeEvent) (PendingResult, error)) error
}
```

`ChangeEvent` is a new backend-neutral struct carrying the decoded fields the service needs: `ProjectID` (private), `BucketName`, `ObjectKey`, `EventName`, `TotalPlainSize`, `StreamID`, `Version`, `CommitTimestamp`. The name `ChangeEvent` avoids collision with `notification.EventRecord`, which is the S3-shaped output struct produced later by the service.

`PendingResult` moves from the `changestream` package to the `eventing` package as part of this refactor. `SpannerEventSource` and `TiDBEventSource` both return `eventing.PendingResult`; `changestream` imports it from there instead of defining it.

- **`SpannerEventSource`** — wraps `changestream.Processor`, decodes each `DataChangeRecord` into a `ChangeEvent` (refactoring the existing `ConvertModsToEvent` + `ProcessRecord` logic).
- **`TiDBEventSource`** — decodes each outbox row into a `ChangeEvent` and calls `fn`. Implements the two-goroutine outbox polling loop described below.

#### Outbox Worker

`TiDBEventSource` implements `EventSource`. It runs as the same `satellite changestream` command. The adapter selection happens in `satellite/mud.go`, which currently provides `changestream.Adapter` by type-asserting `metabase.Adapter` to `changestream.Adapter` and panicking if it is not Spanner. This needs to be replaced with a conditional that provides a `SpannerEventSource` for Spanner adapters and a `TiDBEventSource` for TiDB adapters.

The worker uses two goroutines connected by a buffered channel of capacity 1 to pipeline database reads with Pub/Sub publishing, matching the throughput of the Spanner `partitionDrainer` approach. While one batch is awaiting Pub/Sub confirmation, the reader goroutine is already fetching the next batch. A capacity of 1 keeps at most two batches in memory at once (one being published, one pre-fetched) while still providing the full pipeline benefit.

**Reader goroutine** — polls the outbox and sends batches to the publisher:

```
lastSeenID = 0  # in-memory only; resets to 0 on restart
loop:
  rows = SELECT id, project_id, bucket_name, object_key, version,
                stream_id, total_plain_size, event_type
         FROM bucket_eventing_outbox
         WHERE id > lastSeenID
         ORDER BY id
         LIMIT $batchSize  # tidb-batch-size, default 100

  if no rows:
    sleep $pollInterval  # tidb-poll-interval, default 100ms
    continue

  lastSeenID = rows[last].id
  send rows → batchCh  # buffered(1): blocks only if publisher has two batches already
```

**Publisher goroutine** — decodes each batch into `ChangeEvent`s, calls `fn` (the service), and deletes confirmed rows:

```
loop:
  batch = receive from batchCh

  pendingResults = []
  for each row in batch:
    event = ChangeEvent{
        ProjectID:       row.project_id,
        BucketName:      row.bucket_name,
        ObjectKey:       row.object_key,
        EventName:       row.event_type,
        TotalPlainSize:  row.total_plain_size,
        StreamID:        row.stream_id,
        Version:         row.version,
        CommitTimestamp: row.created_at,
    }
    result, err = fn(event)  # fn is eventing.Service: filters, resolves project ID, publishes
    if err != nil: return err
    # fn returns ImmediateResult (not nil) for skipped events (no config, filtered out)
    pendingResults.append((row.id, result))

  confirmedIDs = []
  for each (id, result) in pendingResults:
    if err = result.Get(ctx); err != nil: return err
    confirmedIDs.append(id)

  DELETE FROM bucket_eventing_outbox WHERE id IN (confirmedIDs)
```

On startup, `lastSeenID` resets to 0, so any rows not yet deleted (unconfirmed delivery from the previous run) are re-read and re-published, fulfilling at-least-once delivery. No cursor state needs to be persisted.

The channel between the goroutines provides natural backpressure: if Pub/Sub is slow, the publisher goroutine stalls on `result.Get()`, the channel fills, and the reader blocks on `send rows → batchCh` until the publisher catches up.

**Error handling**: if `result.Get()` returns an infrastructure error (e.g. context cancellation, transient Pub/Sub failure) mid-batch, the publisher goroutine returns the error immediately. The batch `DELETE` is not executed, so all rows in the batch remain in the outbox. Both goroutines exit, the worker restarts, `lastSeenID` resets to 0, and all undeleted rows — including any that were already confirmed earlier in the same batch — are re-published. This is correct under at-least-once delivery. User-configuration errors (e.g. deleted topic, missing permissions) are handled inside `publisher.Publish()` and cause `result.Get()` to return nil, so they do not abort the batch.

#### Translating Outbox Rows to S3 Events

`TiDBEventSource` decodes each outbox row into a `ChangeEvent` (shown in the pseudocode above) and passes it to the service via `fn`. The service then applies filtering, resolves the private project ID to the public project ID via `CachedPublicProjectIDs.GetPublicID()`, and constructs the S3 `notification.EventRecord`. The existing `notification.go` types (`Event`, `EventRecord`, `EncodeForS3Event`, `MatchEventType`, `MatchFilters`) are reused without modification.

The mapping from outbox columns to `ChangeEvent` fields is one-to-one with no encoding or decoding — the SQL driver scans binary columns directly into the same Go types the struct carries. The service maps `ChangeEvent` to `notification.EventRecord` as follows:

| `ChangeEvent` field | `notification.EventRecord` field |
|---|---|
| `CommitTimestamp` | `eventTime` |
| `EventName` | `eventName` (prefixed with `s3:`) |
| `BucketName` | `s3.bucket.name`, `s3.bucket.arn` |
| `ProjectID` (resolved to public) | `s3.bucket.ownerIdentity.principalId` |
| `ObjectKey` | URL-encoded → `s3.object.key` |
| `TotalPlainSize` | `s3.object.size` |
| `StreamID` + `Version` | `NewStreamVersionID(Version, StreamID)` hex-encoded → `s3.object.versionId` |
| `CommitTimestamp` (Unix nanos) | 16-char uppercase hex → `s3.object.sequencer` |

The `ConvertModsToEvent()` function in `notification.go` is **not used** for the TiDB path — it is Spanner-specific (`spanner.NullJSON`, transaction tag inference) and moves into `SpannerEventSource`.

#### Impact on the Spanner Path

`changestream.Processor`, `MetadataBatcher`, `partitionDrainer`, `bucket_eventing_metadata`, `TransmitEvent`, and `ExcludeTxnFromChangeStreams` are all unchanged. The Spanner backend is wrapped in `SpannerEventSource` which implements the new `EventSource` interface — a mechanical refactor with no behavioral change.

The `notification.go` dependency on `spanner.NullJSON` and `cloud.google.com/go/spanner` moves into `SpannerEventSource`, keeping it isolated from the shared service logic and from the TiDB path.

#### Relation to Phase 2 and Later Changes

The TiDB implementation inherits all current bucket eventing features:

- `bucket_eventing_configs` table (in satellite TiDB) is shared and unchanged.
- `PutBucketNotificationConfiguration` / `GetBucketNotificationConfiguration` S3 API endpoints work as designed in Phase 2.
- Bucket eventing is available to all users — project-level gating (`bucket-eventing.projects`) was removed after Phase 2.
- In-memory LRU caching for bucket notification configs is reused.
- `shouldTransmitEvent()` is reused without modification.
- Satellite-managed encryption requirement is enforced.
- Async Pub/Sub delivery via the `eventing.PendingResult` interface is reused — `TiDBEventSource` calls `Publish` (non-blocking) and confirms delivery via `result.Get()` before deleting the outbox row.

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

The private project ID must never appear in outbox rows written to logs, in event notification messages, or in error messages. The worker resolves the private project ID to the public project ID using the same `CachedPublicProjectIDs` wrapper as the Spanner path before constructing the `notification.EventRecord`.

### Observability

The two-goroutine pipeline has three potential bottlenecks: the outbox read, the Pub/Sub publish, and the outbox delete. Each should be independently observable.

**End-to-end lag** — `created_at` of the oldest unprocessed row (`SELECT created_at FROM bucket_eventing_outbox ORDER BY id LIMIT 1`) vs. current time. A growing lag indicates the worker is falling behind regardless of which stage is the bottleneck. Equivalent to the "lagging watermark" metric from the Spanner path.

**Outbox read latency** — monkit `mon.Task()` on the reader's SELECT query. Spikes here point to TiDB being the bottleneck (slow query, lock contention, or large table scan).

**Pub/Sub publish latency and outcome** — reuse the existing `publish_success` and `publish_failed` eventkit events from `pubsub.go`, which already include end-to-end publish latency (time from `Publish()` call to `result.Get()` returning). Spikes here point to Pub/Sub being the bottleneck.

**Outbox delete latency** — monkit `mon.Task()` on the `DELETE WHERE id IN (...)` call. Spikes here point to TiDB write throughput being the bottleneck.

**Pipeline pressure** — the batch channel capacity between the two goroutines. When the channel is full, the reader is blocked waiting for the publisher to catch up (Pub/Sub or delete is the bottleneck). When the channel is empty, the publisher is blocked waiting for the reader (TiDB read is the bottleneck). Report the current channel fill level as a gauge metric.

### Test plan

Non-exhaustive test plan:

- Verify that no outbox rows are written if no bucket has eventing enabled.
- Verify that an outbox row is written for each relevant metainfo operation when a bucket has eventing enabled.
- Verify that the event type stored in the outbox matches the operation (Put vs Copy vs Delete vs DeleteMarkerCreated).
- Verify that the worker correctly translates outbox rows to S3 event notifications and publishes to Pub/Sub.
- Verify at-least-once delivery: stop the worker, perform uploads and deletes, restart the worker, and confirm all events are delivered without loss.
- Verify that outbox entries are deleted only after successful Pub/Sub confirmation.
- Verify that removing a bucket notification config causes no further outbox writes for that bucket.
- Verify that the Spanner-based eventing path produces identical output before and after the `SpannerEventSource` refactor.

### Rollout

The TiDB eventing path is selected automatically based on the configured metabase adapter. The `bucket_eventing_outbox` table is added as a new step in `TiDBAdapter.TiDBMigration()`.

Two new fields are added to `eventing.Config` for the TiDB path:

| Flag | Default | Description |
|---|---|---|
| `--eventing.tidb-poll-interval` | `100ms` | How long the reader goroutine sleeps when the outbox is empty. |
| `--eventing.tidb-batch-size` | `100` | Maximum number of outbox rows fetched per SELECT. |

### Rollback

Disable the `satellite changestream` service for the TiDB metainfo database variant. Outbox rows accumulate in TiDB during downtime and are processed on restart.

## Out of scope

- Exactly-once delivery.
- TiCDC or Kafka-based CDC.
- Event types beyond those defined in Phases 1 and 2.
- Multiple destinations per bucket.
