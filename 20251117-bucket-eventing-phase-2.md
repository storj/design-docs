tags: [ "satellite", "satellite/metainfo", "bucket eventing", "s3 api" ]
---

# Bucket Eventing Phase 2: S3 API Configuration

## Essentials

### Header

Date: 2025-11-17

Owner: [Kaloyan Raev](https://github.com/kaloyan-raev)

Accountable:
- [Michał Niewrzał](https://github.com/mniewrzal)

Consulted:
- [Maximillian von Briesen](https://github.com/mobyvb)
- [Jennifer Li Johnson](https://github.com/jenlij)
- [Márton Elek](https://github.com/elek)
- [Michael Ferris](https://github.com/ferristocrat)
- [Ivan Fraixedes](https://github.com/ifraixedes)

Informed:
- [#eng-object-storage-eventing](https://storj.slack.com/archives/C0939NQNFFG)

### Context

Phase 1 of bucket eventing introduced basic event notification capabilities using configuration-based enablement (via `bucket-eventing.buckets` config). This required manual configuration by the engineering team for each bucket.

Phase 2 will provide a self-service experience by implementing S3-compatible API endpoints for managing bucket notification configurations, enabling customers to enable/disable eventing and configure filtering rules through standard S3 tools.

Access to the bucket eventing feature will still be gated, but at the project level instead of the bucket level. A new `bucket-eventing.projects` configuration will replace the Phase 1 `bucket-eventing.buckets` configuration. Once a project is enabled, users within that project can use the S3 API to self-manage notification configurations for any of their buckets.

### Goals

Deliver S3-compatible bucket notification configuration API to allow customers to self-manage bucket eventing.

- S3-compatible `PutBucketNotificationConfiguration` and `GetBucketNotificationConfiguration` APIs
- Project-level gating: Projects must be explicitly enabled for bucket eventing via satellite configuration
- Per-bucket notification configuration stored in the satellite database
- Support for filtering rules (prefix and suffix filters)
- Support for event type selection (allow users to choose which event types they want)
- Validation via test event: `PutBucketNotificationConfiguration` must successfully publish a test event to the destination before accepting the configuration
- Google Pub/Sub as the only supported destination type (one configuration per bucket)
- Only available for projects with satellite-managed encryption (path encryption disabled)
- No automated migration path from Phase 1 configuration-based approach

### Non-Goals

- Multiple destinations per bucket
- Complex filter rules (Phase 2 supports only 1 prefix + 1 suffix filter per configuration)
- Destination types other than Google Pub/Sub (no SNS, SQS, Lambda, webhooks)
- Automated migration from Phase 1 `bucket-eventing.buckets` configuration to database storage
- Automated Pub/Sub topic creation or management
- Advanced event types beyond the Phase 1 set

### Approach / Design

#### Configuration Storage

We will explore two approaches for storing bucket notification configurations:

##### Approach A: JSONB Column in bucket_metainfos Table

Store the notification configuration as a JSONB column in the existing `bucket_metainfos` table.

**Pros:**
- Simple schema - single column addition to existing table
- Collocated with other bucket metadata (versioning, object lock)
- Easy to retrieve configuration alongside bucket metadata
- Leverages PostgreSQL/CockroachDB/Spanner JSON support

**Cons:**
- Less queryable - can't easily query by configuration attributes (e.g., find all buckets with a specific topic)
- Complex JSON schema validation needed in application code
- More difficult to add indexes on configuration attributes
- JSONB handling varies across database backends (PostgreSQL vs CockroachDB vs Spanner)

**Schema:**
```sql
ALTER TABLE bucket_metainfos
ADD COLUMN notification_config JSONB;
```

**Example JSON structure:**
```json
{
  "id": "ObjectEvents",
  "topicName": "projects/storj-developer-team/topics/bucket-eventing",
  "events": ["s3:ObjectCreated:Put", "s3:ObjectCreated:Copy", "s3:ObjectRemoved:Delete"],
  "filter": {
    "prefix": "logs/",
    "suffix": ".jpg"
  }
}
```

##### Approach B: Dedicated bucket_eventing_configs Table

Create a new table specifically for notification configurations.

**Pros:**
- Structured, queryable schema with proper types
- Easy to add indexes and constraints
- Clear separation of concerns
- Consistent across all database backends
- Future-proof for adding more complex configurations
- Leverages Spanner default value functions (GENERATE_UUID, CURRENT_TIMESTAMP) for automatic field generation

**Cons:**
- Additional table to manage
- More database migrations
- Additional database query overhead during metainfo operations (CommitObject, DeleteObject, etc.) to fetch notification configuration for TransmitEvent evaluation. Mitigated by Redis caching strategy.

**Schema:**
```sql
CREATE TABLE bucket_eventing_configs (
    project_id       BYTES(MAX) NOT NULL,
    bucket_name      BYTES(MAX) NOT NULL,
    config_id        STRING(MAX) NOT NULL DEFAULT (GENERATE_UUID()),
    topic_name       STRING(MAX) NOT NULL,
    events           ARRAY<STRING(128)> NOT NULL,
    filter_prefix    BYTES(1024),
    filter_suffix    BYTES(1024),
    created_at       TIMESTAMP NOT NULL DEFAULT (CURRENT_TIMESTAMP()),
    updated_at       TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp = TRUE),
    CONSTRAINT bucket_eventing_configs_bucket_fkey
        FOREIGN KEY (project_id, bucket_name)
        REFERENCES bucket_metainfos (project_id, name)
        ON DELETE CASCADE
) PRIMARY KEY (project_id, bucket_name);
```

**Notes:**
- `config_id`: User can optionally provide an ID in the API request (any string value). If not provided, Spanner auto-generates a UUID using `GENERATE_UUID()`.
- `PRIMARY KEY (project_id, bucket_name)`: Ensures only one notification configuration per bucket (Phase 2 limitation).
- **Foreign Key with CASCADE DELETE**: When a bucket is deleted from `bucket_metainfos`, the notification configuration is automatically deleted from `bucket_eventing_configs`. This prevents a new bucket with the same name from inadvertently inheriting the old bucket's notification configuration. Follows the same pattern used throughout the codebase (e.g., `api_keys`, `project_members`, `project_invitations`).

**Decision:** We will use **Approach B** (dedicated table) for the implementation. This provides better queryability, consistency across database backends, and future-proofing for more complex configurations.

#### Project-Level Gating

Only projects explicitly enabled for bucket eventing can use the notification configuration APIs.

A new satellite configuration value `bucket-eventing.projects` will accept a comma-separated list of project UUIDs:

```
bucket-eventing.projects=4d0452d9-395d-498d-927f-a54a6544764e,7a2f8c3b-1e4d-4a5f-9c6b-2d3e4f5a6b7c
```

This configuration will be checked at two points:
1. **API Level**: `PutBucketNotificationConfiguration` and `GetBucketNotificationConfiguration` endpoints will verify the project is in the enabled list. If the project is not enabled, return error code `rpcstatus.Unimplemented` (501 Not Implemented).
2. **Transaction Level**: When determining whether to set `TransmitEvent: true` for metabase operations

The Phase 1 `bucket-eventing.buckets` configuration is **replaced** by this new project-level configuration and will no longer be used. No automated migration will be provided - Phase 1 customers will need to manually reconfigure using the new API.

#### Satellite-Managed Encryption Requirement

Bucket eventing requires satellite-managed encryption to ensure object keys in notifications are unencrypted (or at least decryptable by the satellite).

The `PutBucketNotificationConfiguration` endpoint will check:
- `project.PathEncryption != nil && *project.PathEncryption == false`

If this check fails, return:
- Error code: `rpcstatus.FailedPrecondition`
- Message: `"Bucket eventing requires satellite-managed encryption (path encryption must be disabled)"`

**Note on Encrypted Path Support:**
While the underlying architecture can handle both encrypted and unencrypted object paths, this phase focuses exclusively on supporting unencrypted paths (satellite-managed encryption). This is expected to be the preferred configuration for customers who require bucket eventing capabilities. Support for encrypted paths could be added in a future phase if customer demand warrants the additional complexity.

#### Metainfo API Endpoints

We will add new gRPC methods to the metainfo service for managing bucket notification configurations.

##### SetBucketNotificationConfiguration

```protobuf
message SetBucketNotificationConfigurationRequest {
    RequestHeader header = 1;
    bytes name = 2;  // bucket name
    NotificationConfiguration configuration = 3;
}

message SetBucketNotificationConfigurationResponse {}

message NotificationConfiguration {
    string id = 1;  // configuration ID (e.g., "ObjectEvents")
    string topic_name = 2;  // Pub/Sub topic (projects/PROJECT_ID/topics/TOPIC_ID)
    repeated string events = 3;  // e.g., ["s3:ObjectCreated:Put", "s3:ObjectRemoved:Delete"]
    FilterRule filter = 4;
}

message FilterRule {
    string prefix = 1;  // e.g., "logs/"
    string suffix = 2;  // e.g., ".jpg"
}
```

**Filter Handling:**
- If `filter` field is omitted or nil: No filtering, all objects match
- Empty string for `prefix` or `suffix` is treated as "not set" (no filter applied for that field)
- Both prefix and suffix empty/omitted means no filtering (all objects match)

**Implementation:**

Following S3 behavior, this operation **replaces the entire notification configuration** for the bucket. Each call overwrites any existing configuration.

1. Verify project is enabled for bucket eventing via `bucket-eventing.projects` config
2. Verify project has satellite-managed encryption (path encryption disabled)
3. Check macaroon permissions: `macaroon.ActionPutBucketNotificationConfiguration`
4. Validate the configuration:
   - Topic name format: `projects/PROJECT_ID/topics/TOPIC_ID`
   - Events: Each event must be one of the specific types `["s3:ObjectCreated:Put", "s3:ObjectCreated:Copy", "s3:ObjectRemoved:Delete", "s3:ObjectRemoved:DeleteMarkerCreated"]` OR a valid wildcard `["s3:ObjectCreated:*", "s3:ObjectRemoved:*"]`
   - Filter: At most 1 prefix and 1 suffix
   - If validation fails, return error code `rpcstatus.InvalidArgument` with descriptive message
5. Send test event to the Pub/Sub topic:
   - Create temporary publisher
   - Publish S3 `s3:TestEvent` notification (see format in Appendix)
   - Wait synchronously for publish success (timeout: 10 seconds)
   - If test event fails to publish, return error code `rpcstatus.FailedPrecondition` with the underlying Pub/Sub error message
6. Store configuration in database using UPSERT (`INSERT OR UPDATE`) - replaces any existing configuration for this bucket
7. Return success

**Empty Configuration Handling:**
Following S3 API convention, sending an empty `NotificationConfiguration` (or omitting the `configuration` field) will **delete** the bucket's notification configuration:
- Remove the row from `bucket_eventing_configs` table
- Subsequent `GetBucketNotificationConfiguration` calls will return an empty configuration
- Event notifications for this bucket will stop being delivered (future transactions will have `TransmitEvent: false`)

##### GetBucketNotificationConfiguration

```protobuf
message GetBucketNotificationConfigurationRequest {
    RequestHeader header = 1;
    bytes name = 2;  // bucket name
}

message GetBucketNotificationConfigurationResponse {
    NotificationConfiguration configuration = 1;
}
```

**Implementation:**
1. Verify project is enabled for bucket eventing via `bucket-eventing.projects` config
2. Check macaroon permissions: `macaroon.ActionGetBucketNotificationConfiguration`
3. Query database for notification configuration
4. If no configuration exists, return empty `NotificationConfiguration` (not an error)
5. Return the configuration

**Note:** This endpoint only returns configurations set via the API. It does **not** show Phase 1 config-based setups (from `bucket-eventing.buckets`).

#### Macaroon Permissions

Following the pattern used for Object Lock and Versioning, we will add new macaroon actions:

```go
const (
    ActionPutBucketNotificationConfiguration Action = "PutBucketNotificationConfiguration"
    ActionGetBucketNotificationConfiguration Action = "GetBucketNotificationConfiguration"
)
```

These actions will be checked in the metainfo endpoints before processing the requests.

API keys must have the appropriate permissions to call these endpoints. This follows the S3 model where `s3:PutBucketNotification` and `s3:GetBucketNotification` are separate permissions.

#### Libuplink Integration

The libuplink library (https://github.com/storj/uplink) needs to expose the new metainfo gRPC methods. These will be used primarily by the S3 gateway to implement the S3 notification configuration API.

**New Functions to Add:**

```go
// In private/bucket package (not public API)
func SetBucketNotificationConfiguration(ctx context.Context, project *uplink.Project, bucketName string, config *metaclient.BucketNotificationConfiguration) error
func GetBucketNotificationConfiguration(ctx context.Context, project *uplink.Project, bucketName string) (*metaclient.BucketNotificationConfiguration, error)
```

**Private Package Placement:**

These methods will be added to the **private package** in libuplink because:
- They are primarily intended for S3 gateway use (not general public API)
- Users access bucket eventing through the S3 API (AWS CLI, SDK, etc.), not directly via libuplink

#### S3 Gateway Integration

The S3 gateway uses a fork of Minio (https://github.com/storj/minio) where we add custom methods to the ObjectLayer interface. Following the pattern used for Object Lock (see commit [97cae2c](https://github.com/storj/minio/commit/97cae2c0d7f119ad4e04d73c181b406b7f03d36d)), we need to add similar methods for bucket notifications.

**Minio Fork Changes:**

The Storj Minio fork (`pkg/event/arn.go`) uses a GCP-specific ARN format for Pub/Sub topics:
- Format: `arn:gcp:pubsub::PROJECT_ID:TOPIC_ID`
- Example: `arn:gcp:pubsub::my-gcp-project:my-bucket-events`

This ARN format is used in the S3 XML API for `<Topic>` elements. The gateway-ST converts between this ARN format and the fully-qualified Pub/Sub topic name (`projects/PROJECT_ID/topics/TOPIC_ID`) used internally by the satellite.

**ObjectLayer Interface Methods to Add:**

Add these methods to `cmd/object-api-interface.go` in the Storj Minio fork:

```go
GetBucketNotificationConfig(ctx context.Context, bucket string) (*event.Config, error)
SetBucketNotificationConfig(ctx context.Context, bucket string, config *event.Config) error
```

These follow the exact naming pattern as the Object Lock methods:
- `GetObjectLockConfig(ctx context.Context, bucket string) (*objectlock.Config, error)`
- `SetObjectLockConfig(ctx context.Context, bucket string, config *objectlock.Config) error`

**Implementation in Gateway-ST:**

In the Storj gateway-ST (https://github.com/storj/gateway-st), implement these ObjectLayer methods:

For `SetBucketNotificationConfig`:
1. Receive `*event.Config` from Minio framework (already parsed from S3 XML)
2. Validate the config contains only Pub/Sub topic destinations (reject SNS, SQS, Lambda)
3. Validate at most one topic destination (reject multiple topics)
4. Convert Minio `*event.Config` to metainfo `NotificationConfiguration` protobuf
5. Call `metainfo.SetBucketNotificationConfiguration` gRPC method
6. Wait for synchronous response (including test event validation)
7. Return error or nil

For `GetBucketNotificationConfig`:
1. Call `metainfo.GetBucketNotificationConfiguration` gRPC method
2. Convert metainfo protobuf response to Minio `*event.Config` struct
3. Return the config (Minio framework will serialize to S3 XML)

**No Caching:**
The S3 gateway will **not** cache notification configurations. Each request will query metainfo for the current configuration. This ensures consistency and avoids cache invalidation complexity.

**Implementation in Gateway-MT/Edge:**

The multi-tenant S3 gateway (https://github.com/storj/edge) uses gateway-ST as a dependency. The `GetBucketNotificationConfig` and `SetBucketNotificationConfig` methods must also be implemented in the Edge gateway, delegating to the underlying gateway-ST implementation.

#### TransmitEvent Integration

When metabase operations are called (e.g., `CommitObject`, `DeleteObject`), the metainfo endpoint must determine whether to set `TransmitEvent: true`.

**Decision Logic:**
1. Check if project is enabled via `bucket-eventing.projects` config
2. If not enabled, set `TransmitEvent: false` (exclude from change stream)
3. If enabled, check cache for notification configuration (query database on cache miss)
   - If cache retrieval fails (including database query failures): set `TransmitEvent: true` (fail safe - let eventing service decide), log error, and continue with object operation
4. If no configuration exists, set `TransmitEvent: false`
5. If configuration exists, check if operation's event type is in the configuration's events list
6. If event type doesn't match, set `TransmitEvent: false`
7. If the event type matches, evaluate filtering:
   - Get object key from `streamID.EncryptedObjectKey` (should be unencrypted with satellite-managed passphrase)
   - Check if object key matches prefix filter (if set)
   - Check if object key matches suffix filter (if set)
8. If all checks pass, set `TransmitEvent: true`

**Filter Evaluation:**
- **Prefix filter**: Object key must start with the prefix (e.g., `logs/` matches `logs/2025-01-17.log` but not `archive/logs/file.txt`)
- **Suffix filter**: Object key must end with the suffix (e.g., `.jpg` matches `photo.jpg` but not `photo.jpg.bak`)
- **Both filters**: Object key must match both (AND logic)

**Event Type Evaluation:**

For each metabase operation, determine which event type must be enabled in the configuration:

Requires `s3:ObjectCreated:Put`:
- `CommitInlineObject`
- `CommitObjectWithSegments`

Requires `s3:ObjectCreated:Copy`:
- `FinishCopyObject` - checks only the destination bucket
- `FinishMoveObject` - checks both source and destination buckets (OR logic), since a move affects two locations

Requires `s3:ObjectRemoved:Delete`:
- `DeleteObjectsAllVersions`
- `DeleteObjectExactVersion`
- `DeleteObjectLastCommitted` (unversioned)

Requires `s3:ObjectRemoved:DeleteMarkerCreated`:
- `DeleteObjectLastCommitted` (versioned)

#### Bucket Eventing Service Changes

The bucket eventing service processes change stream records:
1. Reads change stream records
2. Checks cache for current notification configuration (query database on cache miss)
3. Evaluates event type and filter rules (acts as secondary filter after metainfo's primary filter)
4. Transforms matching records to S3 event format
5. Publishes to configured Pub/Sub topics

**Destination Resolution:**
When processing a change record, the service checks the cache for the notification configuration to determine the Pub/Sub destination. If no configuration exists, log warning and discard (this shouldn't happen due to TransmitEvent filtering).

**Configuration Change Handling:**
Due to independent caching in metainfo and eventing services, configuration changes may result in temporary inconsistencies:
- A change stream record may be created (TransmitEvent=true with old config) but filtered out by eventing service (with new config)
- This is expected behavior during the cache TTL window (up to 5 minutes) and results in no event being published
- The warning log "no configuration exists" may appear during this transition period

**Error Handling - Infrastructure Failures:**
When processing a change record fails (config lookup, publisher creation, or event publishing):
1. Log error with change stream record details (project_id, bucket_name)
2. Retry with exponential backoff: 1s, 2s, 4s, 8s (max wait 8s)
3. After exhausting the backoff schedule, emit `bucket_eventing_publish_critical` eventkit event (alerts on-call) and continue retrying at max backoff interval
4. **Do NOT advance change stream position** until the record is successfully processed

This approach:
- ✅ Preserves at-least-once delivery guarantee
- ✅ Prevents silent event loss during infrastructure outages
- ✅ Blocks change stream partition (appropriate for infrastructure failure)
- ✅ Triggers alerting for immediate remediation
- ❌ Partition processing stalls during Redis+DB outage (acceptable trade-off)

**Rationale:** Infrastructure failures (Redis and database both unavailable) require immediate operational response. Silently dropping events would break delivery guarantees. Blocking the partition ensures no data loss and forces resolution of the underlying infrastructure issue.

#### Test Event Format

When `PutBucketNotificationConfiguration` validates the destination, it sends a test event following S3's `s3:TestEvent` format:

```json
{
  "Service": "Storj S3",
  "Event": "s3:TestEvent",
  "Time": "2025-01-17T10:30:00.000Z",
  "Bucket": "my-bucket"
}
```

**Notes:**
- This is a **synchronous** operation - the API call waits for publish confirmation
- Uses a temporary Pub/Sub publisher (not cached)
- Timeout: 10 seconds
- If publish fails (network error, auth error, topic doesn't exist), return error to user
- In production, `metainfo.bucket-eventing-service-account` must be configured with the `bucket-eventing` service account email that customers grant access to their Pub/Sub topics (see Phase 1 "Service Account" section). The test event publisher uses GCP service account impersonation to publish to the topic.

#### Filtering Rules

Phase 2 supports filtering rules following the [S3 notification filtering model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-how-to-filtering.html):

- Up to 1 prefix filter per configuration
- Up to 1 suffix filter per configuration
- Both prefix AND suffix can be specified (both must match)
- Only 1 configuration per bucket (future phases may support multiple configurations like S3)

**Examples (filters are case-sensitive):**
- Prefix `logs/` matches: `logs/2025.txt`, `logs/archive/old.log`
- Prefix `logs/` does NOT match: `Logs/file.txt`, `archive/logs/file.txt`, `mylogs.txt`
- Suffix `.jpg` matches: `photo.jpg`, `dir/image.jpg`
- Suffix `.jpg` does NOT match: `photo.JPG`, `photo.jpeg`, `image.jpg.bak`
- Prefix `logs/` + Suffix `.txt` matches: `logs/file.txt`, `logs/dir/doc.txt`
- Prefix `logs/` + Suffix `.txt` does NOT match: `logs/file.TXT`, `logs/file.log`, `data/file.txt`

**Filter Validation:**
- Maximum length: 1024 characters for both prefix and suffix (consistent with S3)
- Filters are literal strings - wildcards (*, ?) are matched literally, not as patterns
- No validation that prefix/suffix are "sensible" - user can set any string

#### Event Type Selection

Unlike Phase 1 (all events enabled), Phase 2 allows users to select which event types they want to receive.

**Supported Event Types:**
- `s3:ObjectCreated:Put`
- `s3:ObjectCreated:Copy`
- `s3:ObjectRemoved:Delete`
- `s3:ObjectRemoved:DeleteMarkerCreated`

**Wildcard Support (consistent with S3):**
Users can specify category-level wildcards:
- `s3:ObjectCreated:*` → expands to `[s3:ObjectCreated:Put, s3:ObjectCreated:Copy]`
- `s3:ObjectRemoved:*` → expands to `[s3:ObjectRemoved:Delete, s3:ObjectRemoved:DeleteMarkerCreated]`

**Storage:**
Store the event list exactly as specified by the user (including wildcards). The event type evaluation logic expands wildcards at runtime when filtering events.

**Examples:**
- User specifies: `["s3:ObjectCreated:*"]` → Stored in DB as-is
- User specifies: `["s3:ObjectCreated:Put", "s3:ObjectRemoved:Delete"]` → Stored in DB as-is

#### Configuration Caching

To avoid querying the database on every metainfo operation, bucket notification configurations will be cached using Redis, reusing the existing Redis instance used by live accounting (`satellite/accounting/live`).

**Cache Implementation:**

Use Redis with the same pattern as `redisLiveAccounting`:

```go
type redisEventingConfigCache struct {
    client *redis.Client
}
```

**Cache Key:**
- Format: `bucket-eventing:{project_id}:{bucket_name}` (using hex encoding for binary project_id)
- Example: `"bucket-eventing:550e8400-e29b-41d4-a716-446655440000:my-bucket"`
- Prefix ensures no conflicts with live accounting keys

**Cache Configuration:**
- TTL: 5 minutes (as safety net)
- Explicit invalidation on configuration changes
- Reuses existing Redis instance (no additional infrastructure)

**Usage Pattern:**

In metainfo endpoint (for `TransmitEvent` evaluation during object operations):
```go
// Try Redis cache first
config, err := redis.Get(ctx, cacheKey)
if err == redis.Nil {
    // Cache miss - query database
    config, err = endpoint.metabase.GetBucketNotificationConfig(ctx, projectID, bucketName)
    if err == nil {
        // Store in Redis with TTL
        redis.Set(ctx, cacheKey, config, 5*time.Minute)
    }
}
```

In eventing service (for filtering):
```go
// Same pattern as metainfo endpoint
config, err := redis.Get(ctx, cacheKey)
if err == redis.Nil {
    config, err = service.metabase.GetBucketNotificationConfig(ctx, projectID, bucketName)
    if err == nil {
        redis.Set(ctx, cacheKey, config, 5*time.Minute)
    }
}
```

**NOT used for `GetBucketNotificationConfiguration` API endpoint:**

The `GetBucketNotificationConfiguration` endpoint should **always query the database directly** (not use Redis cache) to ensure users see their most recent configuration changes with zero delay.

**Caching Empty Configurations:**

When no notification configuration exists for a bucket in the `bucket_eventing_configs` table, the database query (`GetBucketNotificationConfig`) returns an empty `NotificationConfiguration` (not an error). This empty configuration is cached in Redis just like any other configuration. This prevents repeated database queries for buckets without eventing enabled, which is the common case.

**Explicit Cache Invalidation:**

When `PutBucketNotificationConfiguration` is called (including when setting an empty configuration to delete):
```go
redis.Del(ctx, cacheKey)
```

This invalidates the cache **across all pods immediately** since Redis is shared. The next operation on any pod will fetch fresh configuration from the database.

**Benefits:**
- **Immediate cache invalidation across all pods**: Using Redis enables immediate cache invalidation when configuration changes, solving the UX problem where users enable eventing and expect immediate results
- **Reduces database load**: Spanner queries are expensive; Redis cache provides fast lookups
- **Shared cache**: All metainfo and eventing pods share the same cache, ensuring consistency
- **Reuses existing infrastructure**: No new Redis instance needed (reuses live accounting Redis)
- **Automatic expiration**: 5-minute TTL acts as safety net in case of invalidation failures

**Trade-offs:**
- **Additional network call**: Redis lookup adds latency compared to local LRU cache (but still much faster than Spanner)
- **Performance degradation when Redis unavailable**: Falls back to direct database queries when Redis is unavailable, increasing latency and Spanner load
- **TTL window**: Configuration changes may take up to 5 minutes to propagate if cache invalidation fails (rare edge case)

#### Configuration Consistency

Configuration changes (PUT requests) take effect for **new transactions only**, not in-flight transactions.

**Implementation:**
- Metainfo endpoints read the notification configuration at the **start** of each transaction (from cache)
- Use that configuration for the entire transaction (to set `TransmitEvent` flag)
- Configuration changes made during a transaction do not affect that transaction
- No retroactive changes to change stream records already generated

This approach provides predictable behavior: the configuration active at transaction start determines event generation. S3 documentation does not explicitly specify the behavior for in-flight operations during configuration changes.

## Disclaimers

### Alternatives Considered

**Alternative 1: Support Migration from Phase 1**
We considered automatically migrating existing `bucket-eventing.buckets` configurations to the database. Rejected because:
- Adds complexity to the initial rollout
- Phase 1 has limited customers (can be migrated manually)
- Clean separation between phases is simpler

**Alternative 2: Cache Notification Configurations in S3 Gateway**
We considered caching notification configurations in the S3 gateway to reduce metainfo queries. Rejected because:
- Adds cache invalidation complexity
- Configuration changes are infrequent (not a performance bottleneck)
- Simpler to always query metainfo for consistency

**Alternative 3: Local LRU Cache per Pod**
We considered using local in-memory LRU cache (like rate limiting uses) instead of Redis. Rejected because:
- Poor UX: Users enabling eventing would wait up to 5 minutes to see events (cache TTL window)
- No cross-pod cache invalidation: Each pod has independent cache, leading to inconsistent behavior
- Example bad UX: Create bucket → upload object (caches empty config) → enable eventing → wait 5 minutes
- Redis solves this with immediate cross-pod cache invalidation

**Alternative 4: Async Test Event**
We considered making the test event asynchronous (return success immediately, validate in background). Rejected because:
- User experience is worse (errors discovered later)
- S3 API convention is synchronous validation
- Test event should be fast (<1 second typically)

### Open Questions

- Should we add rate limiting to `PutBucketNotificationConfiguration` to prevent abuse?
  - Current answer: Not adding rate limiting in Phase 2. May consider in a future phase if abuse becomes a problem.
- How do we handle Pub/Sub quota exhaustion after the test event succeeds?
  - Current answer: Same as Phase 1 - log errors, track metrics, continue processing

## Reminders

### Security / Privacy

#### Private Project ID

The private project ID is a secret. Ensure it is never leaked in:
- Event notification messages (use public project ID)
- Logs
- API responses
- Error messages

#### Pub/Sub Topic Validation

The test event validates that:
- The topic exists
- The topic is accessible by the `bucket-eventing` service account
- The topic accepts messages

However, it does NOT validate:
- Pub/Sub quotas (customer may hit quota limits later)
- Pub/Sub subscription configuration (customer responsible for setting up subscriptions)

#### Macaroon Permissions

New API keys created for bucket eventing should have the minimum necessary permissions:
- `ActionPutBucketNotificationConfiguration` for enabling/disabling eventing
- `ActionGetBucketNotificationConfiguration` for viewing configuration
- Standard bucket read/write permissions for data operations

### Observability

Extend Phase 1 metrics with:
- **Configuration Count**: Track total number of active notification configurations

### Migration from Phase 1

**No automatic migration** will be provided. Phase 1 customers will need to enable bucket eventing using the S3 API (`PutBucketNotificationConfiguration`).

The `bucket-eventing.buckets` configuration will be **removed** and replaced by the self-service API. It will not work alongside Phase 2 configurations.

### Test Plan

Non-exhaustive test plan:

**Configuration API:**
- Test `PutBucketNotificationConfiguration` with valid configuration succeeds
- Test `PutBucketNotificationConfiguration` with invalid topic name fails
- Test `PutBucketNotificationConfiguration` with unreachable topic fails (test event fails)
- Test `PutBucketNotificationConfiguration` for project without satellite-managed encryption fails
- Test `PutBucketNotificationConfiguration` for project not in `bucket-eventing.projects` fails
- Test `PutBucketNotificationConfiguration` with empty configuration deletes the config
- Test `GetBucketNotificationConfiguration` returns correct configuration
- Test `GetBucketNotificationConfiguration` for bucket with no config returns empty (not error)
- Test API key without `ActionPutBucketNotificationConfiguration` permission is rejected
- Test API key with bucket-specific caveat can only configure that bucket (other buckets rejected)

**Filtering:**
- Test events with prefix filter - matching objects trigger events, non-matching don't
- Test events with suffix filter - matching objects trigger events, non-matching don't
- Test events with both prefix and suffix - only objects matching both trigger events
- Test events with no filter - all objects trigger events
- Test edge cases: empty prefix, empty suffix, Unicode characters

**Event Type Selection:**
- Test configuration with only `s3:ObjectCreated:*` - only create events sent
- Test configuration with only `s3:ObjectRemoved:*` - only delete events sent
- Test configuration with single specific event type - only that event sent
- Test configuration with multiple event types - all specified events sent

**Integration:**
- Test S3 gateway ObjectLayer methods (`GetBucketNotificationConfig`/`SetBucketNotificationConfig`) work correctly
- Test multiple buckets with different configurations
- Test configuration changes take effect for new operations
- Test TransmitEvent correctly set based on configuration and filters

**Test Event:**
- Test event published successfully to valid topic
- Test event fails for invalid topic
- Test event fails for topic without permissions
- Test event timeout after 10 seconds

## Out of Scope

- Multiple destinations per bucket
- Destination types other than Google Pub/Sub
- Migration tool/script from Phase 1 to Phase 2
- Automatic Pub/Sub topic creation
- Pub/Sub subscription management
- Event batching (still one event per notification)
- Additional event types beyond Phase 1 set
- UI for managing notification configurations (API-only for Phase 2)
- Billing/cost tracking for bucket eventing usage

## Appendix

### S3 PutBucketNotificationConfiguration XML Example

Example request body for S3 `PUT /?notification`:

```xml
<NotificationConfiguration>
    <TopicConfiguration>
        <Id>MyNotificationConfig</Id>
        <Topic>arn:gcp:pubsub::my-gcp-project:my-bucket-events</Topic>
        <Event>s3:ObjectCreated:*</Event>
        <Event>s3:ObjectRemoved:Delete</Event>
        <Filter>
            <S3Key>
                <FilterRule>
                    <Name>prefix</Name>
                    <Value>logs/</Value>
                </FilterRule>
                <FilterRule>
                    <Name>suffix</Name>
                    <Value>.txt</Value>
                </FilterRule>
            </S3Key>
        </Filter>
    </TopicConfiguration>
</NotificationConfiguration>
```

Empty configuration (to delete):
```xml
<NotificationConfiguration />
```

### Test Event Example

```json
{
  "Service": "Storj S3",
  "Event": "s3:TestEvent",
  "Time": "2025-01-17T10:30:00.000Z",
  "Bucket": "my-bucket"
}
```

### Protobuf Definition

Full protobuf definitions for the new endpoints:

```protobuf
// Bucket notification configuration management
service Metainfo {
    rpc PutBucketNotificationConfiguration(PutBucketNotificationConfigurationRequest) returns (PutBucketNotificationConfigurationResponse);
    rpc GetBucketNotificationConfiguration(GetBucketNotificationConfigurationRequest) returns (GetBucketNotificationConfigurationResponse);
}

message PutBucketNotificationConfigurationRequest {
    RequestHeader header = 1;
    bytes name = 2;  // bucket name
    NotificationConfiguration configuration = 3;  // omit or empty to delete
}

message PutBucketNotificationConfigurationResponse {}

message GetBucketNotificationConfigurationRequest {
    RequestHeader header = 1;
    bytes name = 2;  // bucket name
}

message GetBucketNotificationConfigurationResponse {
    NotificationConfiguration configuration = 1;  // empty if no config exists
}

message NotificationConfiguration {
    string id = 1;  // configuration ID (e.g., "ObjectEvents")
    string topic_name = 2;  // fully-qualified Pub/Sub topic
    repeated string events = 3;  // S3 event types
    FilterRule filter = 4;  // optional filter rules
}

message FilterRule {
    string prefix = 1;  // prefix filter (optional)
    string suffix = 2;  // suffix filter (optional)
}
```
