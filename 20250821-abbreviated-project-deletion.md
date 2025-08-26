tags: [ "satellite", "satellite/console", "self-serve project deletion" ]
---

# Abbreviated Project Deletion Flow

## Essentials

Date: 2025-08-21

Owner: wilfred-asomanii

Accountable:
- Console team

### Context

The current process of deleting a project in the Satellite UI is very involving of the users, requiring them to manually
delete data we could otherwise delete automatically. This is what happens when a user attempts to delete a project:

1. They are prompted to delete the project's data/buckets.
2. They are prompted to delete the project's accesses (API keys, access grants, etc.).
3. Then, if the user is `paid` (`user.kind == 1`), either:
    - They are prompted to pay for outstanding billable usage for the current billing period, or
    - They are exempt if usage is less than a configured `payments.delete-project-cost-threshold`
4. Finally, the user goes through verification steps so that account can be deleted (password, MFA if enabled and email verification)

The problem with this flow is that it is a cumbersome multistep process, some of which could be automated.
Also, by the time the user is able to delete the project, they would've already deleted all the project's data which makes
it effectively deleted.

This document describes a new flow that requires minimal user interaction while ensuring that project deletion is still safe and secure.

### Approach / Design

For this new flow to be technically feasible, a few changes are required:
1. **New project status**: Introduce a new project status `ProjectPendingDeletion` (2) to indicate that a project is pending deletion.
2. **New column in `projects` table**: Add a new column `status_updated_at (timestamp)` to track when the project status was last updated.
3. **New chore for project deletion**: Implement a background chore that will handle the actual deletion of projects marked as pending deletion.

#### Enhanced User Flow

**New Project Deletion Flow:**

1. **Pre-deletion Validation**: Before anything, we will check the project's billing status:
    - If the user is `paid` and the project usage is above `payments.delete-project-cost-threshold`, prevent deletion
    - Display clear error message explaining billing requirements

2. **User Verification Steps**: Once validation passes, also verify that the user is authorized to delete the project:
    - Authenticate action with user's password
    - Require MFA if enabled
    - Confirm the user email via verification code

   This is the last step the user will be required to perform.

3. **Automated Cleanup**: After successful verification,
    - Update project status to `ProjectPendingDeletion` (`status` = 2)
    - Return success response to user immediately
    - Project deletion chore handles actual data deletion

#### New Project Status
Currently, projects have the following statuses:
- `ProjectDisabled` (0): Project deleted or disabled. Usage is not billed for this project.
- `ProjectActive` (1): Active project

To support the new deletion flow, we will introduce a new status:
- `ProjectPendingDeletion` (2): This marks the project for automatic deletion after a grace period, after which the status
    will be updated to `ProjectDisabled`. This will allow us to be able to recover the project if needed. In the same way
    as `ProjectDisabled`, usage is not billed for this project.

#### Database Change - New Column
A new nullable column will be added to the `projects` table:
```sql
ALTER TABLE projects ADD COLUMN status_updated_at TIMESTAMP WITH TIME ZONE;
```
This column will track whenever a project's status is updated, which is important for the deletion chore to know when to delete the project.
Also, a composite index will be added on `(status, status_updated_at)` to optimize the chore's query.
```sql
ALTER TABLE projects ADD INDEX idx_status_updated_at (status, status_updated_at);
```

#### Project Deletion Chore
A new chore `ProjectDeletionChore` will be implemented to handle the actual deletion of projects marked as pending deletion.
This functionality can also be added to an existing chore that is concerned with deleting data for users marked for deletion.
This chore will run periodically and perform the following steps:
1. Query for projects with status `ProjectPendingDeletion` and `status_updated_at` older than a configured grace period,
    ordered by `status_updated_at` to process the oldest marked projects first. This query will also get projects whose deletion
    failed in a previous run, so that they can be retried until they are marked as `ProjectDisabled`.
2. For each project:
     - Delete all project buckets and their objects/segments (via metabase)
     - Delete all API keys and access grants
     - Update project status to `ProjectDisabled`.

The chore will have the following configurations:

```go
type Config struct {
    Enabled            bool          // whether the chore is enabled
    Interval           time.Duration // how often to run the chore
    ListLimit          int           // how many projects to query in a batch
    MaxConcurrency     int           // how many delete workers to run concurrently
    GracePeriod        time.Duration // how long to wait before deleting a project after marking it for deletion
}
```

### Admin Functionality
- Admin will be able to mark a project as `ProjectPendingDeletion` via the admin API and UI.
- Admin will be able to take a project out of `ProjectPendingDeletion` status and revert it back to `ProjectActive` if needed.
  - Provided that the grace period has not elapsed.

### Open questions

- Should we provide an admin API to manually trigger cleanup of specific projects?
- Should the grace period be configurable per project or user tier?

## Reminders

### Security / Fail safes

- User verification steps (password, MFA, email) must be completed before marking project as `ProjectPendingDeletion`
- Marked projects within the grace period must remain inaccessible to users but preserved for potential recovery

### Logging

- Log user ID and project ID when marking project for deletion
- Deletion chore start/completion
- Count and log the number of projects deleted after a chore run cycle.
- Count and log the number of failed deletion attempts.
- Log the project IDs and user IDs for all deletion operations.
- Log the amount of data deleted for each project if possible.

### Test plan

Tests for this design should confirm:

- Pre-deletion validation prevents deletion when billing threshold exceeded
- User verification steps are required before project status change
- Project status is immediately updated to `ProjectPendingDeletion` after verification
- `ProjectDeletionChore` only deletes projects older than grace period

### Rollout / Rollback

This flow will be disabled by default. A new configuration will be added to toggle this flow and the chore.
The flow will only be enabled if the configuration is set to true while the chore will be enabled only if the new flow
is enabled AND the chore's enabled configuration (which is false by default) is set to true.

When the flow is disabled, the old flow will be used as before. In case there is an issue with the new flow, the
configuration can be set to false, disable the new flow and the chore. Projects marked as pending deletion will remain
in status `ProjectPendingDeletion` indefinitely until the chore is re-enabled or manually cleaned up.
