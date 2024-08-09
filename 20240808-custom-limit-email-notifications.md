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

### Goals

1. Give users advance notice that they are getting close to their custom limits (80% used)
2. Inform users when they have exceeded their custom limits (100% used)

### Approach / Design

Things we want to avoid:
* unintentionally sending multiple emails per "threshold event" (e.g. storage usage on project abc has exceeded 80% of the custom limit)
* failing to send an email for a "threshold event" (e.g. satellite restart at the same time as threshold event, resulting in dropped communications)

These constraints require that we make some DB updates.

#### Config Updates

A single feature flag should be added on the satellite to enable "limit threshold email notifications". It should default to be turned off, until it is turned on in production.

#### DB Updates

For each project, we need to track whether emails were sent for each "threshold event":

* storage usage exceeded 80% of custom storage limit
* storage usage exceeded 100% of custom storage limit
* bandwidth usage exceeded 80% of custom bandwidth limit
* bandwidth usage exceeded 100% of custom bandwidth limit

We could add a new column to the projects table for each of these, but we can also minimize the schema change by using the bits of a single int field for each one. Example of one approach accomplishing this via a Go struct:

```golang
type ProjectNotificationFlags struct {
	StorageExceeded80  bool
	StorageExceeded100 bool
	EgressExceeded80   bool
	EgressExceeded100  bool
}

const (
	storage80Filter  int = 1      // 00...0001
	storage100Filter int = 1 << 1 // 00...0010
	egress80Filter   int = 1 << 2 // 00...0100
	egress100Filter  int = 1 << 3 // 00...1000
  /*
    example db values:
    3 = b11 = storage 80 and storage 100 are true
    5 = b101 = storage 80 and egress 80 are true
    15 = b1111 = all four are true
  */
)

func ProjectNotificationFlagsFromInt(dbVal int) *ProjectNotificationFlags {
	p := &ProjectNotificationFlags{}
	if storage80Filter&dbVal > 0 {
		p.StorageExceeded80 = true
	}
	if storage100Filter&dbVal > 0 {
		p.StorageExceeded100 = true
	}
    ...
	return p
}

func (p *ProjectNotificationFlags) Int() int {
	toReturn := 0
	if p.StorageExceeded80 {
		toReturn |= storage80Filter
	}
	if p.StorageExceeded100 {
		toReturn |= storage100Filter
	}
    ...
	return toReturn
}
```

If we use this approach, we only need to add a single updatable int column to the projects table, and since int has 32 bits, we would also be able to support any other similar flags we might want in the future, without schema changes.

#### Event detection and email sending in a chore

We will add a new chore to the satellite core, with the goal of detecting "threshold events" and sending emails.

The chore would:

* for each project on the satellite:
    * if project does not have any custom limits set, skip
    * query current project usage
    * for each custom limit type (storage, egress):
        * if project does not have custom limit for this type, skip
        * for each threshold (80%, 100%):
             * if project usage for this limit type has exceeded threshold:
                 * if email was already sent for this "threshold event", skip
                 * send email for this threshold event, and update project flags in DB to indicate an email was sent
            * if project usage for this limit type has *not* exceeded threshold:
                * if email was set for this threshold event, reset the value to indicate "unsent" (this means that we have entered a new billing period or the user has reset their custom limits)

The chore could have multiple "workers" to allow parallelization, and how often the chore runs would be configurable.

Pros:

* Only requires adding "email sent" flags to `projects` DB table - simple schema change
* The entire feature being on the satellite core avoids race conditions that are introduced by depending on metainfo/API pods

Cons:

* Timeliness of emails depends on how often the chore runs - see "alternatives considered" for more info about why we did not choose a more "responsive" method

#### Summary

* Updating projects table schema to have flags indicating when emails were sent
* Adding a chore on the satellite core to detect when emails need to be sent, send emails, and keep flags pertaining to project notifications in the DB up to date

## Disclaimers

### Anti-goals

* impacting metainfo performance
* sending excessive limit notifications (only one email for 80% and one for 100%, per limit per project)
* failing to send a limit notification when the user passes the 80% or 100% threshold
* sending emails to users who have not set custom limits

### Alternatives considered

1. Use an existing chore on the satellite core rather than adding a new one
    * tally chore is a possible candidate
    * mixes up email notification logic with other features
2. Use third-party tools, and send emails completely outside the satellite
    * it would not use the same email sender as other satellite emails to the user
    * could run on non-prod DB, which wouldn't impact satellite, but would introduce some delay (however often the backup gets updated)
    * not dependent on satellite release cycle (quicker to set up/test/adjust)
    * more access/specialized knowledge required for maintenance - satellite code is more "accessible"
    * likely requires two separate third-parties - one to do queries, and one to send emails. If we go with existing tools for these, the task my require cross-team work with datascience and/or marketing.
3. Detect "threshold events" during metainfo validation, at the time of usage
    * emails would not be delayed due to chore intervals; they would be sent close to when the 80% or 100% threshold was exceeded
    * could potentially have an impact on metainfo performance
    * requires an additional database table (e.g. `project_events`) to track queue of events to send emails for
    * race condition introduced with multiple API pods: events need to be deduplicated to avoid sending two emails for the same event
    * still requires a central core service outside the API pod to handle processing the email queue and sending emails

### Open questions

* should thresholds be configurable on the satellite? Or are we okay "hardcoding" to 80% and 100%?
* what future improvements might we want to make to this feature, and does the current design inhibit that? E.g. will we ever want the user to be able to customize the thresholds for which they will get emailed?

### Test plan

Test cases:

* for each limit type (storage, egress):
    * no custom limit: no email should be received, even when limit is hit
    * custom limit * 80% < current usage < custom limit: one 80% email should be received; no duplicates
    * custom limit <= current usage: one 100% email should be received; no duplicates
    * after email is received, increase custom limit so that threshold is no longer met. When new threshold is met, a new email should be sent
    * after email is received, remove custom limit. No new emails should be received

### Rollout

After we verify the emails work as intended in QA, we simply need to turn the feature on in production.

After the feature is turned on in production, we can change the feature to be enabled by default.

### Rollback

If there are any issues with this feature, we simply have to turn the feature flag off to stop sending emails. These notifications are not an essential feature, so it is okay to turn it off for an extended period of time.

## Out of scope
