tags: ["satellite", "emails", "limits", "notifications"]
---

# Project Limit Notification Emails 

## Essentials

Users are able to set custom egress and storage limits on their project. They also have a separate segment limit, which cannot be customized without going through support. The goal of this design is to send users with custom bandwidth/storage limits emails when they hit 80% and 100% usage of their custom limits, and to send the same kinds of emails for segment limits. This addition ensures that a user is aware that they are approaching their current limits, and gives them the opportunity to increase their limits before exceeding them.

The initial rollout of this feature will be opt-in per project.

### Header

Date: 2024-07-26 

Owner: mobyvb

Accountable:
- Console Team

Consulted:
-

Informed:
-

### Context

Pro users are currently able to set customized project limits for storage and egress. Right now, the only way for a user to check how close they are to their limits is to go to the project dashboard in the satellite UI. A user could also learn about exceeding their limits when uploads or downloads begin failing, but there is no advance notice in this case.

The purpose of "Project Limit Notification Emails" is to inform users automatically when they are getting close to, or exceeding their custom limits.

In addition to notifications about custom limits, we want to notify users when they get close to or exceed their segment limits.

### Goals

1. Give users advance notice that they are getting close to their custom limits (80% used)
2. Inform users when they have exceeded their custom limits (100% used)

### Approach / Design

**Things we want to avoid**
* unintentionally sending multiple emails per "threshold event" (e.g. storage usage on project abc has exceeded 80% of the custom limit)
* failing to send an email for a "threshold event" (e.g. satellite restart at the same time as threshold event, resulting in dropped communications)
* too long of a delay between occurrence of event and notification (e.g. >30m is probably too long, and 30m may even be too long)
* significantly impacting metainfo performance

#### Config Updates

A feature flag should be added on the satellite to enable "limit threshold email notifications". It should default to be turned off, until it is turned on in production. This flag enables/disables:
1. threshold event detection and queueing in the satellite API
2. threshold event processing and email sending in the satellite core

We need a configuration for the "buffer time" between event occurrence and email sending. This buffer time allows us to deduplicate instances of the same event queued by multiple satellite API pods.

```
limit-email-notifications.enabled: false
limit-email-notifications.email-time-buffer: 10m
```

#### DB Updates

For each project, we need to track whether limit notification emails are enabled for that project. This allows us to implement opting in or out of email notifications on the project settings page of the satellite UI. We could decide to have a single flag to enable/disable all notifications for the project, or add one flag per limit type (storage, bandwidth, segment), providing the user with more granular control.

We also need to track whether emails were sent for each "threshold event":

* storage usage exceeded 80% of custom storage limit
* storage usage exceeded 100% of custom storage limit
* bandwidth usage exceeded 80% of custom bandwidth limit
* bandwidth usage exceeded 100% of custom bandwidth limit
* segment usage exceeded 80% of segment limit
* segment usage exceeded 100% of segment limit

We could add a new column to the projects table for each of these (enabled flags + threshold events), but we can also minimize the schema change by using the bits of a single int field for each one.

If we use this approach, we only need to add a single updatable int column to the projects table, and since int has 32 bits, we would also be able to support other similar threshold events in the future, without schema changes:

```
// project contains information about a user project.
model project (
    ...
    // notification_flags is a bit string indicating whether certain limit-related events are enabled
    // or have occurred
    field notification_flags int ( updatable )
```

The way flags are ordered in `notification_flags` is not important, as long as the order/meaning of particular bits does not change. Example bit definitions:

```
00000001	custom storage limit notifications enabled
00000010	custom storage limit 80% threshold
00000100	custom storage limit 100% threshold
00001000	custom bandwidth limit notifications enabled
00010000	custom bandwidth limit 80% threshold
00100000	custom bandwidth limit 100% threshold
...
```

In addition to indicating which emails are enabled/have been sent in the `projects` table, we also need to track project events in a queue, requiring a new table. The design and code for this table and its usage is similar to the existing `node_events` table, used for sending storage nodes emails based on reputation-related events:
```
// project_event table contains a project event queue for events which require an email notification.
model project_event (
	key id

	index (
		name project_id_created_at_last_attempted_index
		fields project_id created_at last_attempted
		where project_event.email_sent = null
	)

	// id is a UUID for this event.
	field id             blob
	// project_id is the project associated with this event.
	field project_id     blob
	// event is the event kind - it can be used as a mask for `project.notification_flags`
	field event          int
	// created_at is when this event was added.
	field created_at     timestamp ( default current_timestamp )
	// last_attempted is when nodeevents chore last tried to send the email.
	field last_attempted timestamp ( nullable, updatable )
	// email_sent when the email sending succeeded.
	field email_sent     timestamp ( nullable, updatable )
)

```

The `event` field here is an int, which can be set to the bit definition associated with the event (e.g. `b00000010` = storage exceeded 80% threshold). The type serves a dual purpose as a bitwise filter and enum value so that we do not need to maintain a separate mapping of event identifier -> bit filter.

#### Event detection in metainfo:

The general idea of event detection in metainfo is to perform an additional check during limit validation on new uploads and downloads. Here, we are detecting two main categories of events:
1. Before the upload/download, the threshold criteria was not met, and afterwards, the threshold criteria is met:
    * enqueue an event to the project notifications queue indicating that an email should be sent to the user
2. The threshold criteria was met at some point in the past (as indicated by project flags), but after the upload/download, the threshold criteria is not met:
    * enqueue an event to the project notifications queue indicating that the notifications flag should be reset for the relevant threshold 

More specifically, 

When limit validation occurs for an upload:
* if limit emails are disabled globally, or disabled for the project, skip
* for segment and custom storage limits:
  * for each limit threshold (starting at highest threshold):
    * if this upload causes usage to meet or cross threshold:
      - enqueue `(projectID, thresholdEventKind)` to `project_events`
      - skip checking lower thresholds
	* if usage is below the threshold, but the project is flagged indicating that the threshold was exceeded:
      - enqueue `(projectID, thresholdEventKind)` to `project_events` ("threshold event kind" would indicate to reset the flag with no notification)

When limit validation occurs for a download:
* if limit emails are disabled globally, or disabled for the project, skip
* for each limit threshold (starting at highest threshold):
  * if this download causes egress to meet or cross custom egress threshold: 
    - enqueue `(projectID, thresholdEventKind)` to `project_events`
    - skip checking lower thresholds
  * if egress usage is below the custom threshold, but the project is flagged indicating that the threshold was exceeded:
    - enqueue `(projectID, thresholdEventKind)` to `project_events` ("threshold event kind" would indicate to reset the flag with no notification)

While the intended behavior for event detection is straightforward, implementation may be tricky. As an example, metainfo validates whether upload limits would be exceeded for a new object in the metainfo package [here](https://github.com/storj/storj/blob/8d9fab8825e92c4c77252e3891530e5d04c430eb/satellite/metainfo/validation.go#L574) - calling a function `ExceedsUploadLimits` in accounting/projectusage.go [here](https://github.com/storj/storj/blob/b552501c39352ba80487b6991975e8d13f549410/satellite/accounting/projectusage.go#L125). The project usage service retrieves the current storage and segment usage from a [redis cache](https://github.com/storj/storj/blob/b552501c39352ba80487b6991975e8d13f549410/satellite/accounting/live/redis.go#L66) in accounting/live/redis.go.

Our current live accounting project usage behavior is designed to allow for very quickly getting and updating a handful of values per project (specific limits and corresponding usage) with minimal DB calls.

We need to have additional information during limit validation to properly queue events: the `notification_flags` flags from the `projects` table. This is because we not only need to check (1) is current usage above or below the threshold? We also need to check (2) are notification emails for this threshold enabled? And (3) has a notification been triggered for this threshold before?

This additional information could be merged into the live accounting redis cache. This would likely be the most optimal/efficient, but at the risk of changing the responsibilities of the live accounting package.

If we don't add additional information to the live accounting cache (or some other cache), it requires making a query to the `projects` table for the `notification_flags` - something we want to avoid to keep metainfo as efficient as possible.

#### Queue processing in core:

Create a chore on the satellite (or update existing console emailreminders chore):
* loop:
  * select all unprocessed events for one project in the `project_events` table, which were created before `now-emailTimeBuffer`, where `emailTimeBuffer` comes from the satellite config.
    * for "threshold exceeded"-type events:
      * deduplicate events: send only one email per event per threshold
	  * if `project.notification_flags` indicates these types of emails are disabled, mark `project_events` rows as processed and proceed with no further action
      * check if emails have been sent for these threshold events in `project.notification_flags`. If they have, mark as processed or delete relevant rows from `project_events` without sending an email. If they haven't, send email(s) to project owner, mark `project_events` row as processed, and update `project.threshold_events` accordingly
	* for "reset threshold"-type events:
	  * check if `project.threshold_events` indicates that a notification was sent for this limit type. If it has, reset it to indicate "not sent". If the value is already reset, discard/mark the `project_events` row as processed, and proceed with no further action.
	
#### Satellite UI Updates

In the satellite UI, we want to inform users about these notification email types, and make it easy to enable/disable emails for projects.

**Project Settings** - add toggles on project settings page allowing user to turn on/off segment limit, custom storage limit, and custom egress limit emails.

**Edit Project Limits** - add toggle or checkbox to "edit storage limit" and "edit bandwidth limit" dialogs to enable/disable email notifications when editing limits.

**Project Dashboard** - add banner or some other link informing users of the new feature, and how to turn it on

#### Summary

* Update projects table schema to have flags indicating which emails are enabled and which emails were sent
* Add "project events" table to track unsent limit threshold events
* Update metainfo limit validation to queue limit threshold events to the DB
* Add a chore on the satellite core to process limit threshold event queues: deduplicate events for each project, send emails for the events (if applicable), and update the DB accordingly

## Disclaimers

### Anti-goals

* significantly impacting metainfo performance
* sending excessive limit notifications (only one email for 80% and one for 100%, per limit per project)
* failing to send a limit notification when the user passes the 80% or 100% threshold
* sending storage/bandwidth emails to users who have not set custom limits
* significant delays between event occurrence and notification

### Alternatives considered

1. Add or use an existing chore on the satellite to detect events and trigger emails
    * fewer edge cases to consider in detection logic due to avoiding concurrency
    * maximum delay for email is the chore duration. E.g. for a chore that runs every 24 hours, maximum delay is 24h. For 1h chore, maximum delay is 1h
    * lots of unnecessary checks for detection. E.g. if we have 1000 projects, we need to check all 1000 every chore loop, even if only 10 projects were active since the previous check
2. Use third-party tools, and send emails completely outside the satellite
    * it would not use the same email sender as other satellite emails to the user
    * could run on non-prod DB, which wouldn't impact satellite, but would introduce some delay (however often the backup gets updated)
    * not dependent on satellite release cycle (quicker to set up/test/adjust)
    * more access/specialized knowledge required for maintenance - satellite code is more "accessible"
    * likely requires two separate third-parties - one to do queries, and one to send emails. If we go with existing tools for these, the task my require cross-team work with datascience and/or marketing.
3. Support monitoring via API
    * notification setup in this case is handled by the user instead of the satellite
    * requires officially supporting a general API intended to be used outside of the satellite UI 

### Open questions

* do we need to make it easy to make users opted-in to notifications by default? Or for an admin to easily turn certain emails on/off for many users?

### Test plan

Test cases:

* for each custom limit type (storage, egress):
    * no custom limit: no email should be received, even when limit is hit
    * custom limit * 80% < current usage < custom limit: one 80% email should be received; no duplicates
    * custom limit <= current usage: one 100% email should be received; no duplicates
    * after email is received, increase custom limit so that threshold is no longer met. When new threshold is met, a new email should be sent
    * after email is received, remove custom limit. No new emails should be received
* for segment limit type:
    * same test cases as storage/egress limits, but since segment limit isn't customizable by the user, these emails are sent for all projects
* test enabling/disabling different email types for a project, and ensure that the emails are sent/not sent appropriately

### Rollout

After we verify the emails work as intended in QA, we simply need to turn the feature on in production.

Once the feature is turned on, users will still need to opt in in order to receive these emails.

### Rollback

If there are any issues with this feature, we simply have to turn the feature flag off to stop sending emails. These notifications are not an essential feature, so it is okay to turn it off for an extended period of time.

## Out of scope
