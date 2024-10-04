tags: ["satellite", "emails", "limits", "notifications"]
---

# Project Limit Notification Emails 

## Essentials

Users are able to set custom egress and storage limits on their project. The goal of this design is to send users with custom limits emails when they hit 80% and 100% usage of their custom limits. This addition ensures that a user is aware that they are approaching their current limits, and gives them the opportunity to increase their limits before exceeding them.

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

For each project, we need to track whether emails were sent for each "threshold event":

* storage usage exceeded 80% of custom storage limit
* storage usage exceeded 100% of custom storage limit
* bandwidth usage exceeded 80% of custom bandwidth limit
* bandwidth usage exceeded 100% of custom bandwidth limit
* segment usage exceeded 80% of segment limit
* segment usage exceeded 100% of segment limit

We could add a new column to the projects table for each of these, but we can also minimize the schema change by using the bits of a single int field for each one. Example of one approach accomplishing this via a Go struct:

```golang
type ProjectNotificationFlags struct {
	StorageExceeded80  bool
	StorageExceeded100 bool
	EgressExceeded80   bool
	EgressExceeded100  bool
	SegmentExceeded80  bool
	SegmentExceeded100 bool
}

// ProjectNotificationKind represents a type of "project notification" event.
// The value can be used as a bit filter to determine if the event kind has occurred for a project.
type ProjectNotificationKind int

const (
	CustomStorage80  ProjectNotificationKind = 1      // 00...0001
	CustomStorage100 ProjectNotificationKind = 1 << 1 // 00...0010
	CustomEgress80   ProjectNotificationKind = 1 << 2 // 00...0100
	CustomEgress100  ProjectNotificationKind = 1 << 3 // 00...1000
	Segment80        ProjectNotificationKind = 1 << 4
	Segment100       ProjectNotificationKind = 1 << 5
)

func ProjectNotificationFlagsFromInt(dbVal int) *ProjectNotificationFlags {
	p := &ProjectNotificationFlags{}
	if CustomStorage80&dbVal > 0 {
		p.StorageExceeded80 = true
	}
	if CustomStorage100&dbVal > 0 {
		p.StorageExceeded100 = true
	}
    ...
	return p
}

func (p *ProjectNotificationFlags) Int() int {
	toReturn := 0
	if p.StorageExceeded80 {
		toReturn |= CustomStorage80
	}
	if p.StorageExceeded100 {
		toReturn |= CustomStorage100
	}
    ...
	return toReturn
}
```

If we use this approach, we only need to add a single updatable int column to the projects table, and since int has 32 bits, we would also be able to support other similar threshold events in the future, without schema changes:

```
// project contains information about a user project.
model project (
    ...
    // threshold_events is a bit string indicating whether certain limit-related events have occurred.
    // Each event is associated with a different 
    field threshold_events int ( updatable )
```

In addition to indicating which emails have been sent in the `projects` table, we also need to track project events in a queue, requiring a new table. The design and code for this table and its usage is similar to the existing `node_events` table, used for sending storage nodes emails based on reputation-related events:
```
// project_event table contains a project event queue for events which require an email notification.
model project_event (
	key id

	index (
		name project_events_id_created_at_index
		fields project_id created_at
		where project_event.email_sent = null
	)

	// id is a UUID for this event.
	field id             blob
	// project_id is the project associated with this event.
	field project_id     blob
	// event is the event kind, an integer mapping to a `ProjectNotificationKind`
	field event          int
	// created_at is when this event was added.
	field created_at     timestamp ( default current_timestamp )
	// last_attempted is when nodeevents chore last tried to send the email.
	field last_attempted timestamp ( nullable, updatable )
	// email_sent when the email sending succeeded.
	field email_sent     timestamp ( nullable, updatable )
)

```

The `event` field here is an int, which can be set to the `ProjectNotificationKind` associated with the event. The type serves a dual purpose as a bitwise filter and enum value so that we do not need to maintain a separate mapping.

#### Event detection in metainfo:

The general idea of event detection in metainfo:

When limit validation occurs for an upload:
* for segment and custom storage limits:
  * for each limit threshold (starting at highest threshold):
    * if this upload causes usage to meet or cross threshold:
      - enqueue `(projectID, thresholdEventKind)` to `project_events`
      - skip checking lower thresholds

When limit validation occurs for a download:
* for each limit threshold (starting at highest threshold):
  * if this download causes egress to meet or cross custom egress threshold: 
    - enqueue `(projectID, thresholdEventKind)` to `project_events`
    - skip checking lower thresholds

TODO: metainfo probably needs to reset `project.threshold_events` when the conditions for threshold events are no longer met (e.g. rolling into a new billing period brings egress usage down to 0).

#### Queue processing in core:

Create a chore on the satellite (or update existing console emailreminders chore):
* loop:
  * select all unprocessed events for one project in the `project_events` table, which were created before `now-emailTimeBuffer`, where `emailTimeBuffer` comes from the satellite config.
    * deduplicate events: send only one email per event per threshold
    * check if emails have been sent for these threshold events in `project.threshold_events`. If they have, mark as processed or delete relevant rows from `project_events` without sending an email. If they haven't, send email(s) to project owner, mark `project_events` row as processed, and update `project.threshold_events` accordingly

#### Summary

* Update projects table schema to have flags indicating when emails were sent
* Add "project events" table to preserve unsent limit threshold events
* Update metainfo limit validation to queue limit threshold events to the DB
* Add a chore on the satellite core to process limit threshold event queues: deduplicate events for each project, send emails for the events, and update the DB to indicate that the emails were sent

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

### Rollout

After we verify the emails work as intended in QA, we simply need to turn the feature on in production.

After the feature is turned on in production, we can change the feature to be enabled by default.

### Rollback

If there are any issues with this feature, we simply have to turn the feature flag off to stop sending emails. These notifications are not an essential feature, so it is okay to turn it off for an extended period of time.

## Out of scope