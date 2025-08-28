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

## Disclaimers

### Anti-goals

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


### Alternatives considered

### Open question

## Reminders

### Security / Privacy

### Observability

### Test plan

### Rollout

### Rollback

## Out of scope
