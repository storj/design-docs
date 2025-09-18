tags: [ "satellite", "satellite/metainfo", "bucket eventing" ]
---

# Bucket Eventing Phase 1

## Essentials

### Header

Date: 2025-08-28

Owner: [Kaloyan Raev](https://github.com/kaloyan-raev)

Accountable:
- [Michał Niewrzał](https://github.com/mniewrzal)

Consulted:
- [Márton Elek](https://github.com/elek)
- [JT Olio](https://github.com/jtolio)
- [Jacob Willoughby](https://github.com/onionjake)
- [David Colantuoni](https://github.com/dcolantuoni)

Informed:
- [#eng-object-storage-eventing](https://storj.slack.com/archives/C0939NQNFFG)

### Context

Media & Entertainment (M&E) customers need a reliable, low-latency way to kick off workflows—e.g., transcoding, QC—when new assets land in their buckets.

### Goals

Deliver an S3-compatible bucket event notifications MVP to a select group of initial customers.

- [At-least-once notification delivery](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-how-to-event-types-and-destinations.html#event-ordering-and-duplicate-events). No guarantee for order or duplications.
- Single destination per bucket
- Google Pub/Sub as the only supported destination type
  - We will ask the customer to provide a Pub/Sub topic in their own GCP account
  - [Pub/Sub Lite](https://cloud.google.com/pubsub/docs/choosing-pubsub-or-lite) is deprecated and we won’t support it
- `s3:ObjectCreated:Put`, `s3:ObjectRemoved:Delete`, and `s3:ObjectRemoved:DeleteMarkerCreated` as the only supported types. All these event types are enabled for the bucket.
- Minimal bucket eventing configuration: the customer opens a support ticket, which is processed by the engineering team
- Notification message body compliant with version 2.1 of the [S3 event message structure](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-content-structure.html)
- User documentation how to configure [Pub/Sub Push subscription](https://cloud.google.com/pubsub/docs/push) to an HTTPS webhook

### Approach / Design

#### COGS Analysis

Assumptions for implementation:
- Bucket events will be sourced from Spanner Change Stream.
- Google Pub/Sub will be the destination for bucket events.
- A custom worker will be used to read the Spanner Change Stream and push notifications to the Pub/Sub topic.

Spanner cost:
- Storage of change records. Every time a data change occurs (insert, update, or delete) that is being watched by a change stream, Spanner writes a data change record to its internal storage. This write happens within the same transaction as the data change itself, ensuring consistency. The charge for the storage consumed by these records is at the same rate as the standard Spanner database storage. The amount of storage used depends on the volume of changes and the data retention period configured for the change stream. The retention period can be from 1 to 30 days.
- No egress free if the change stream reader is in the same region. Otherwise, the standard egress fee applies.

Compute Engine cost:
- The compute cost for reading the change stream and pushing notification to the destination Pub/Sub topic is proportional to the number of change records written to the change stream.
- No egress free if the destination Pub/Sub topic is global or configured to the same region.

Pub/Sub cost:
- Throughput (message volume). The total volume of data published to the topic is charged at $40/TiB. The size of a message is calculated as the sum of its payload and its attributes, with a minimum billable size of 1 KB per message.
- Message storage. Any message retained beyond the default 24 hours is charged at $0.27 per GiB-month.
- Egress fee. The same throughput fee of $40/TiB applies for the message volume delivered by a pull or push subscription. Although the fee is the same, it is applied independently from the fee for publishing to the topic (i.e. the data entering the topic). So the total throughput fee for the volume entering and exiting the topic (if not filtering or transformation is applied) is $80/TiB.

Strategies for reducing the COGS:
- Reduce the number of change records written to the Spanner change stream.
  - Change records should be added to the Spanner change stream only for buckets with eventing enabled, and for the SQL queries that are relevant for bucket events.
  - We can achieve this using transaction-level records exclusion in Spanner.
  - This significantly reduces the cost for the Spanner storage and the Change Stream Reader compute.
- Reduce the number of table columns to be tracked by the Spanner change stream.
- Require the customer to own the Pub/Sub topic in their GCP account.
  - This transfers the throughput cost of $80/TiB to the customer, which is the primary cost for the bucket eventing.
  - This also transfers the responsibility for any inefficiencies in the customer’s webhook to the customer themselves.
  - This is inline with AmazonS3. They don’t push notifications directly to webhooks, but require the use of some other AWS services like SNS, SQS, or Lambda and the customer bears the cost for using them.
  - As a bonus for the customer, they can take advantage of the 10 GiB free tier for throughput. They can also take actions of optimizing their cost through topic/subscription configuration.
- Require the customer Pub/Sub topic to be global or configured to the same region as the Storj satellite.
  - This eliminates the egress fee from the Change Stream Reader to the Pub/Sub topic.
- Minimize the retention period for the Spanner change stream (is the minimum of 1 day enough?).
- Support filtering rules (out of scope for Phase 1)
- Support fine-grain event types (out of scope Phase 1)

#### Implementation

##### Creating the Spanner Change Stream

We will [create a change stream](https://cloud.google.com/spanner/docs/change-streams/manage) on the `objects` table with the following DDL:

```sql
CREATE CHANGE STREAM bucket_eventing
FOR objects (
   status,
   total_plain_size
) OPTIONS (
   exclude_ttl_deletes = TRUE,
   exclude_update = TRUE,
   allow_txn_exclusion = TRUE
);
```

The change stream will watch the following table columns:
- The `project_id`, `bucket_name`, `object_key`, and `version` columns are the primary key and implicitly included in the change records, so we must not specify them. Otherwise, the DDL execution will fail.
- The `status` column to determine if an `INSERT` change record corresponds to a committed object (`status = 3` or `4`) or a delete marker (`status = 5` or `6`).
- The `total_plain_size` column to populate the `size` field of the `s3:ObjectCreated:*` notifications.

We set the following option to reduce the number of change records in the change stream:
- The `exclude_ttl_deletes = TRUE` option, because TTL deletes correspond to `s3:LifecycleExpiration:*` events, which are out of scope for Phase 1.
- The `exclude_update = TRUE` option, because we are not interested in them. An object is committed to the table with a `DELETE`+`INSERT` transaction instead of with an `UPDATE`.
- The `allow_txn_exclusion = TRUE` option to exclude all transactions with option `exclude_txn_from_change_streams = TRUE` (i.e. all transactions related to buckets that are not enabled for eventing).

As Spanner change streams are created with executing a DDL like above, we can simply add the DDL as a new step to the satellite DB migration.

Later, if we decide to change the list of watched columns or any of the options, we can [modify the change stream](https://cloud.google.com/spanner/docs/change-streams/manage#modify) with another migration step.

We will have only one change stream per Spanner instance to handle bucket events for all customers.

##### Enabling Eventing for a Bucket

For Phase 1 of bucket eventing, we anticipate limited customer usage. To enable this feature with minimal implementation effort, a configuration value like `bucket-eventing.buckets` will be used.

The `bucket-eventing.buckets` config will accept a comma-separated list of `project_id:bucket_name:topic_id` tuples. Each project can have multiple buckets, but each bucket is restricted to a single topic.

An example configuration value is: `4d0452d9-395d-498d-927f-a54a6544764e:my-bucket:projects/storj-developer-team/topics/bucket-eventing`.

Upon startup, the metainfo service will read this configuration and store the configurations in a `map[metabase.BucketLocation]string` variable.

##### Excluding Transactions From the Change Stream

To optimize Spanner costs, it is crucial to [exclude](https://cloud.google.com/spanner/docs/change-streams#transaction-exclusion) all transactions from the change stream except those directly related to bucket eventing.

Before creating the change stream, ensure that the `exclude_txn_from_change_streams` transaction option is applied to all write transactions on the objects table. This option should not be set for read-only transactions, as it will result in an invalid argument error.

The `exclude_txn_from_change_streams` option should only be set to false if a transaction generates an event for a bucket with enabled eventing:
- `CommitInlineObject`
- `CommitObjectWithSegments`
- `FinishCopyObject`
- `DeleteObjectLastCommitted`
- `DeleteObjectExactVersion`

##### Reading From the Change Stream

Spanner offers three methods for reading change streams: Dataflow, the Spanner API, and a Kafka connector.

The Kafka connector is not suitable as we do not have an existing Kafka cluster.

Dataflow is the simplest option, but its Spanner change stream reading functionality is only supported in Java. To avoid introducing another programming language, we will not use Dataflow.

Therefore, we will utilize the [Spanner API](https://cloud.google.com/spanner/docs/change-streams/details#query). A key consideration with this approach is the need to manage change stream partition lifecycles, a feature that Dataflow provides out-of-the-box. We can leverage a [community-maintained library](https://pkg.go.dev/github.com/cloudspannerecosystem/spanner-change-streams-tail/changestreams) to assist with this.

The logic responsible for reading the change stream and publishing messages to Pub/Sub must operate within Google Cloud, specifically in the same region as the satellite’s Spanner instance. Implementing this as a new Metainfo service is the most appropriate solution.

##### Pushing to the Pub/Sub Topic

Currently, we anticipate that each bucket will have its own Pub/Sub topic. However, customers have the flexibility to use a single Pub/Sub topic for multiple buckets if they choose.

In Phase 1, customers will be responsible for creating and managing their Pub/Sub topics within their own GCP accounts.

While it is technically feasible for Storj to create and maintain Pub/Sub topics in our own GCP account, this would require us to monitor usage for billing purposes. Since billing is not part of Phase 1, we will explore Storj-managed Pub/Sub topics in a later phase.

The bucket eventing service will read change records from the Spanner change stream. For each record, it will verify if the project ID and bucket name match a Pub/Sub topic in the configuration. If a match is found, the service will transform the change record (as detailed in the subsequent section) and publish it as a message to the corresponding Pub/Sub topic. Should there be no matching Pub/Sub topic, the service will log the change record as a warning and discard it. It is not anticipated to encounter change records without a matching Pub/Sub topic, as such records should have been filtered out by the Metainfo service.

##### Transforming Change Records to Pub/Sub Messages
The S3 bucket event notifications follow the version 2.1 [event message structure](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-content-structure.html).

```json
{  
   "Records":[  
      {  
         "eventVersion":"2.1",
         "eventSource":"aws:s3",
         "awsRegion":"us-west-2",
         "eventTime":"The time, in ISO-8601 format, for example, 1970-01-01T00:00:00.000Z, when Amazon S3 finished processing the request",
         "eventName":"event-type",
         "userIdentity":{  
            "principalId":"Amazon-customer-ID-of-the-user-who-caused-the-event"
         },
         "requestParameters":{  
            "sourceIPAddress":"ip-address-where-request-came-from"
         },
         "responseElements":{  
            "x-amz-request-id":"Amazon S3 generated request ID",
            "x-amz-id-2":"Amazon S3 host that processed the request"
         },
         "s3":{  
            "s3SchemaVersion":"1.0",
            "configurationId":"ID found in the bucket notification configuration",
            "bucket":{  
               "name":"amzn-s3-demo-bucket",
               "ownerIdentity":{  
                  "principalId":"Amazon-customer-ID-of-the-bucket-owner"
               },
               "arn":"bucket-ARN"
            },
            "object":{  
               "key":"object-key",
               "size":"object-size in bytes",
               "eTag":"object eTag",
               "versionId":"object version if bucket is versioning-enabled, otherwise null",
               "sequencer": "a string representation of a hexadecimal value used to determine event sequence, only used with PUTs and DELETEs"
            }
         }
      }
   ]
}
```

The bucket eventing service will transform the [change record fields](https://cloud.google.com/spanner/docs/change-streams/details#data-change-records) to the message values like this:

- `eventVersion`: will be hardcoded to `2.1`
- `eventSource`: will be hardcoded to `storj:s3`
- `awsRegion`: will be skipped for Phase 1
  - What does Storj equivalent to the AWS region? Is it the placement? If it is, we don’t have it available in the objects table, but only in the segments table. Is it the satellite?
- `eventTime`: will be set to the `commit_timestamp` from the change record
- `eventName`: will be set to one of the supported S3 event type following these rules:
  - `ObjectCreated:Put` if the change record’s `mod_type` is `INSERT`, and the `status` new value is either `CommittedUnversioned(3)` or `CommittedVersioned(4)`. This event type will be also set in those case where it is supposed to have `s3:ObjectCreated:Post`, `s3:ObjectCreated:Copy`, and `s3:ObjectCreated:CompleteMultipartUpload` instead as we currently we don’t have an easy way to distinguish between these event type from the change record.
  - `ObjectRemoved:DeleteMarkerCreated` if the change record’s `mod_type` is `INSERT`, and the `status` new value is either `DeleteMarkerVersioned(5)` or `DeleteMarkerUnversioned(6)`.
  - `ObjectRemoved:Delete` if the change record’s `mod_type` is `DELETE`, and the `status` old value is either `CommittedUnversioned(3)`, `CommittedVersioned(4)`, `DeleteMarkerVersioned(5)`, or `DeleteMarkerUnversioned(6)`.
- `userIdentity`: will be skipped as the `objects` table does not keep information who initiated the database transaction
- `requestParameters`: will be skipped as the `objects` table does not keep information for the source IP of the client who initiated the database transaction
- `requestParameters`: will be skipped as the `objects` table does not keep information for the client request ID and the host that processed the request
- `s3SchemaVersion`: will be hardcoded to `1.0`
- `configurationId`: will be hardcoded to `ObjectEvents`
  - In a future phase, it will be set to the `ID` from the `PutBucketNotificationConfiguration` request that enabled the bucket eventing
- `bucket->name`: will be set to the `bucket_name` key of the change record
- `bucket->ownerIdentity->principalId`: will be set to the public project ID of the bucket. The bucket eventing chore will resolve the public project ID from the `project_id` key (the private project ID) of the change record.
- `bucket->arn`: will be set to `arn:storj:s3:::<bucket_name>`
- `object->key`: will be set to the `object_key` key of the change record. There will be no attempt to decrypt the object key. If the object key is encrypted it will be the user's responsibility to decrypt it. We will provide the tooling and documentation.
- `object->size`: will be set to the new `total_plain_size` value of the change record
- `object->eTag`: will be skipped for Phase 1
- `object->versionId`: will be set to the `version` key of the change record and skipped if the bucket is not versioning-enabled.
- `object->sequencer`:  will be set to the `commit_timestamp` UNIX nanoseconds from the change record, converted to a 16-character, zero-padded, uppercase hex string.

## Disclaimers

### Anti-goals

### Alternatives considered

- We considered the API pods to be the source of bucket events. We concluded that the Spanner change stream makes it easier to guarantee at-lease-once delivery.

### Open question

For Phase 1:
- How are we going to distinguish between `ObjectCreated:Put`, `ObjectCreated:Post`, `ObjectCreated:CompleteMultipartUpload`, and `ObjectCreated:Copy` events from Spanner change records.
- What permissions do we need to set to Spanner and Pub/Sub?

For future phases:
- How do we provide these info to the notifications:
  - `awsRegion`
  - `userIndentity`
  - `sourceIPAddress`
  - `x-amz-request-id`
  - `x-amz-id-2`
  - `eTag`
    - Can the `eTag` be decrypted with the satellite-managed passphrase?


## Reminders

### Security / Privacy

#### Private Project ID

The private project ID is a secret. We have to ensure that we don’t leak any private project ID in the event notification messages or in logs, and always convert it to the public project ID.

#### Google Cloud Platform Permissions

##### Service Account

We must create a new service account for each GCP project where we operate a satellite. The service account should have a descriptive and stable ID, so we can include it in the user documentation. The customer will have to grant permission to this service account to push message to their Pub/Sub topic.

Suggested service account ID: `bucket-eventing`

Service account email address: `bucket-eventing@<project-id>.iam.gserviceaccount.com`

##### Compute Engine VM Instance

The Spanner Change Stream Reader should be deployed to a Google Compute Engine VM instance in the same GCP project of the satellite and the same region of the Spanner instance.

The service account for this VM instance must be set to the `bucket-eventing` service account discussed above.

The `Cloud Platform` access scope must be enabled for this VM instance so it can call the Spanner API (to read the change stream) and the Pub/Sub API (to publish messages).

##### Spanner Instance

We must assign the `Cloud Spanner Database Reader` role to the `bucket_eventing` service account in satellite Spanner instance. This allows the Spanner Change Stream Reader to read the Spanner change stream.

##### Pub/Sub Topic

The customer must assign the `Pub/Sub Publisher` role to the Storj `bucket_eventing` service account in their Pub/Sub topic. This allows the Spanner Change Stream Reader to publish messages to their Pub/Sub topic.

### Observability

The bucket eventing service should report metrics on:
- Number of unexpected change records, indicating that we are not excluding all unnecessary transactions from the change stream.
- Delta between the current time and the timestamp of the change records. Long delays would indicate a performance issue.
- Stats for pushed event messages per bucket. This will help us understand the eventing usage per customer.
  - The stats should include min, max, avg size of the message, which will help us determine the cost if we own the Pub/Sub topics in the future.

### Test plan

There are local Spanner and Pub/Sub emulators that provide basic local testing. Wherever they fall short in functionality, we can use test instances in Google cloud for the integration testing.

Non-exhaustive test plan that emphasizes on a few key points:
- Check that no change records are read from the change stream if no bucket is enabled for eventing.
- Check that an change stream reader logs an error if the event notification cannot be delivered to the Pub/Sub topic.
- Check that change record is properly transformed to event notifications and delivered to the Pub/Sub topic.
  - Check that the object version ID is only included for buckets with versioning enabled.
- Check that no logs include the private project ID. Wherever a project ID is included, it must be the public project ID.
- Configure an HTTP push notificaton to the Pub/Sub topic and check that the events are delivered to the configured HTTP endpoint.
- Configure multiple buckets for eventing and check that the notification are properly delviered to the respective Pub/Sub topic.
- Make multiple uploads and deletes in a few seconds, and check that all event notification are delivered to the Pub/Sub topic within 2 seconds.

## Out of scope

- Multiple destinations per bucket
- Filtering rules
- Event types other than `s3:ObjectCreated:Put`, `s3:ObjectRemoved:Delete`, and `s3:ObjectRemoved:DeleteMarkerCreated`
  - `s3:ObjectCreated:Post`, `s3:ObjectCreated:Copy`, and `s3:ObjectCreated:CompleteMultipartUpload` are out of scope too as we don’t have an easy way to distinguish them from `s3:ObjectCreated:Put`
- Fine-grain configuration of event types, e.g. only `s3:ObjectCreated:*` or only `s3:ObjectRemoved:Delete` event types. All `s3:ObjectCreated:*` and `s3:ObjectRemoved:*` events are always available for a bucket with enabled eventing.
- Event batching. Event notification will contain only one bucket event.
- Some fields from event message structure that are not trivial to provide:
  - `awsRegion`
  - `userIdentity`
  - `requestParameters`
  - `responseElements`
  - `eTag`
- Satellite UI, Admin UI, or S3 API (Get/PutBucketNotificationConfiguration) for configuring bucket eventing. It will be done via a support ticket.
- Billing. We will observe the additional COGS associated with the introduction of bucket eventing, but we will not charge the customers yet.

## Appendix

### Reference AmazonS3 Event Notification Messages

Test notification:

```json
{
  "Service": "Amazon S3",
  "Event": "s3:TestEvent",
  "Time": "2025-08-19T12:06:41.134Z",
  "Bucket": "kaloyanraev-eventing-versioning",
  "RequestId": "Y44M7AEZ4094C205",
  "HostId": "HImzvLC0BD9EOegbJtl0huUpUC0I+Rdtk+oWfVM/ML/D+s8zZQxux8Q8g4bqx+S0ilJICD8PQzM="
}
```

Upload in an unversioned bucket:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2025-08-15T12:48:21.889Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "A17XYCDB5IK9TN"
      },
      "requestParameters": {
        "sourceIPAddress": "212.5.158.124"
      },
      "responseElements": {
        "x-amz-request-id": "7BP7XCW7MEFZSM1D",
        "x-amz-id-2": "lcHSINuv1tq647rD5T+TIF5L4QPPdHNxT0DKM2sKrxBQ9YHgRwKnurQRkul+t9OrNQnFtUKaijBQHeQuvYp9uuUNEL9pO9C6ZxLVzpPCiNI="
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "ObjectEvents",
        "bucket": {
          "name": "kaloyanraev-eventing",
          "ownerIdentity": {
            "principalId": "A17XYCDB5IK9TN"
          },
          "arn": "arn:aws:s3:::kaloyanraev-eventing"
        },
        "object": {
          "key": "storj-logo-mark-color.jpg",
          "size": 64908,
          "eTag": "648edf24a9148598217c55b41c42a84a",
          "sequencer": "00689F2C95DA726DCD"
        }
      }
    }
  ]
}
```

Delete in an unversioned bucket:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2025-08-15T12:56:46.680Z",
      "eventName": "ObjectRemoved:Delete",
      "userIdentity": {
        "principalId": "A17XYCDB5IK9TN"
      },
      "requestParameters": {
        "sourceIPAddress": "212.5.158.124"
      },
      "responseElements": {
        "x-amz-request-id": "1TMRCAHG1NWF4J2X",
        "x-amz-id-2": "gEBXKKm+hPJT04Motcn49Y9Kp0yyD9H0irXYlvqs3hNlCc74cR79dPiDgyXw0r2LzUPeyMTMeZwp99jiFG/+H5LhdJf9cSF91ItwbfA3QQ8="
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "ObjectEvents",
        "bucket": {
          "name": "kaloyanraev-eventing",
          "ownerIdentity": {
            "principalId": "A17XYCDB5IK9TN"
          },
          "arn": "arn:aws:s3:::kaloyanraev-eventing"
        },
        "object": {
          "key": "storj-logo-mark-color.jpg",
          "sequencer": "00689F2E8EA8DF4617"
        }
      }
    }
  ]
}
```

Upload in a versioned bucket:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2025-08-19T12:10:38.000Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "A17XYCDB5IK9TN"
      },
      "requestParameters": {
        "sourceIPAddress": "92.247.48.98"
      },
      "responseElements": {
        "x-amz-request-id": "NJ59W66SNKVSPZNR",
        "x-amz-id-2": "OglZYUaIJQXO68Z1qJmdl/rAkte4AkML/J/jzYPMXFHeLG836kuJhW9psVAKgQOd/VKdbJLYd2mMfWH1QA7n+OuzwPXEScKEXWZ55WIjvZ8="
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "ObjectEvents",
        "bucket": {
          "name": "kaloyanraev-eventing-versioning",
          "ownerIdentity": {
            "principalId": "A17XYCDB5IK9TN"
          },
          "arn": "arn:aws:s3:::kaloyanraev-eventing-versioning"
        },
        "object": {
          "key": "storjv3.pdf",
          "size": 749643,
          "eTag": "ae88c74482d6b23aa5de21a129611762",
          "versionId": "rJiPdgcFaEeV2N2Us6yYU8ObkyJeEL7j",
          "sequencer": "0068A469BDE7041115"
        }
      }
    }
  ]
}
```

Copy in a versioned bucket:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2025-08-19T12:17:49.045Z",
      "eventName": "ObjectCreated:Copy",
      "userIdentity": {
        "principalId": "A17XYCDB5IK9TN"
      },
      "requestParameters": {
        "sourceIPAddress": "92.247.48.98"
      },
      "responseElements": {
        "x-amz-request-id": "4KKH5EYKKD5V4RXW",
        "x-amz-id-2": "WgQyBbdoHj24gbXkvfoJYi3ulC+sluyUomMjtx7BIRpqhnLAEXlcF3R5884/1sJLdwIn+eBdSTu6nTU0A0+c2f/eCKH8CujY3n4Vr6Qrg9g="
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "ObjectEvents",
        "bucket": {
          "name": "kaloyanraev-eventing-versioning",
          "ownerIdentity": {
            "principalId": "A17XYCDB5IK9TN"
          },
          "arn": "arn:aws:s3:::kaloyanraev-eventing-versioning"
        },
        "object": {
          "key": "copy/storjv3.pdf",
          "size": 749643,
          "eTag": "ae88c74482d6b23aa5de21a129611762",
          "versionId": "ggUgyNgN95Fe9pTZ0a7e.O3dDSrOYimf",
          "sequencer": "0068A46B6CE017B360"
        }
      }
    }
  ]
}
```

Create a delete marker in a versioned bucket:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2025-08-19T12:23:05.261Z",
      "eventName": "ObjectRemoved:DeleteMarkerCreated",
      "userIdentity": {
        "principalId": "A17XYCDB5IK9TN"
      },
      "requestParameters": {
        "sourceIPAddress": "92.247.48.98"
      },
      "responseElements": {
        "x-amz-request-id": "VJCDGSTEMAKVRTFG",
        "x-amz-id-2": "hTQbT8ezm1I9ZCewX5MI9847slZo8l5kHqmXqvuWVxigtQa/RO1nq5pUyQBTBv4jlpEeN+0IgIN4WXeau4CICZmD1J4ZIhQRv7LnLS5ms2s="
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "ObjectEvents",
        "bucket": {
          "name": "kaloyanraev-eventing-versioning",
          "ownerIdentity": {
            "principalId": "A17XYCDB5IK9TN"
          },
          "arn": "arn:aws:s3:::kaloyanraev-eventing-versioning"
        },
        "object": {
          "key": "storjv3.pdf",
          "eTag": "d41d8cd98f00b204e9800998ecf8427e",
          "versionId": "86iGIOikTC_a1B1ofO9rfAlEEb1ShcVy",
          "sequencer": "0068A46CA93FF7352F"
        }
      }
    }
  ]
}
```

Delete a delete marker in a versioned bucket:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2025-08-19T12:25:42.043Z",
      "eventName": "ObjectRemoved:Delete",
      "userIdentity": {
        "principalId": "A17XYCDB5IK9TN"
      },
      "requestParameters": {
        "sourceIPAddress": "92.247.48.98"
      },
      "responseElements": {
        "x-amz-request-id": "0S31WTCRNQD1RB4C",
        "x-amz-id-2": "+5fLXYmJJ/NXOUWLJK5K67BmKdz1iNE3KB/GuKrrFDGZhWcLgmr1GXVZaQA4+li2oFLe24IxuZwlY/zosJRJIU/kra9VLxAv"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "ObjectEvents",
        "bucket": {
          "name": "kaloyanraev-eventing-versioning",
          "ownerIdentity": {
            "principalId": "A17XYCDB5IK9TN"
          },
          "arn": "arn:aws:s3:::kaloyanraev-eventing-versioning"
        },
        "object": {
          "key": "storjv3.pdf",
          "versionId": "86iGIOikTC_a1B1ofO9rfAlEEb1ShcVy",
          "sequencer": "0068A46D4609AE53C0"
        }
      }
    }
  ]
}
```

Delete an object version in a versioned bucket:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2025-08-19T12:27:39.942Z",
      "eventName": "ObjectRemoved:Delete",
      "userIdentity": {
        "principalId": "A17XYCDB5IK9TN"
      },
      "requestParameters": {
        "sourceIPAddress": "92.247.48.98"
      },
      "responseElements": {
        "x-amz-request-id": "Q8CGT3CHB5TK1Q1B",
        "x-amz-id-2": "ZlJvF6/xBvrPq0x9vC81OfRM8SnD6G8nx2X3wtpy9kFWBXSIUUXwB6XEIT7XUQtqW9gaLk51vtrmlYup/wvkOpYJAzptLMfa"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "ObjectEvents",
        "bucket": {
          "name": "kaloyanraev-eventing-versioning",
          "ownerIdentity": {
            "principalId": "A17XYCDB5IK9TN"
          },
          "arn": "arn:aws:s3:::kaloyanraev-eventing-versioning"
        },
        "object": {
          "key": "storjv3.pdf",
          "versionId": "rJiPdgcFaEeV2N2Us6yYU8ObkyJeEL7j",
          "sequencer": "0068A46DBBE66B75D0"
        }
      }
    }
  ]
}
```

Multipart upload in a versioned bucket:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2025-08-21T06:01:36.249Z",
      "eventName": "ObjectCreated:CompleteMultipartUpload",
      "userIdentity": {
        "principalId": "A17XYCDB5IK9TN"
      },
      "requestParameters": {
        "sourceIPAddress": "92.247.48.98"
      },
      "responseElements": {
        "x-amz-request-id": "GKT6YKTYN4K8B5GG",
        "x-amz-id-2": "UgtJJqIQXvWeODWeweKdzn9JBduwsYgigUrFZWWkE1kjxb06rKH0Mivox9eWPBrOUP/iXyuodQXVABb5AhD84uG8NbBWiJPT4SFN1V19tiU="
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "ObjectEvents",
        "bucket": {
          "name": "kaloyanraev-eventing-versioning",
          "ownerIdentity": {
            "principalId": "A17XYCDB5IK9TN"
          },
          "arn": "arn:aws:s3:::kaloyanraev-eventing-versioning"
        },
        "object": {
          "key": "bbb_sunflower_1080p_30fps_normal.mp4",
          "size": 276134947,
          "eTag": "e30d2648a6e1c6d43d00f84006340bc2-17",
          "versionId": "RpZ29eNhA3jU1YCUADX6ijqUlZuJ54VB",
          "sequencer": "0068A6B602CA35D4AF"
        }
      }
    }
  ]
}
```
