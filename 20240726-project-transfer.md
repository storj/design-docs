tags: ["satellite", "ui", "project"]
---

# Project Transfer

## Essentials

Users may want to transfer ownership of their project to another account. This may be because the original user wants to delete their account while preserving the project, or that the original user wants to transfer billing responsibilities for the project to another user.

### Header

Date: 2024-07-26 

Owner: mobyvb

Accountable:
- Console team

Consulted:
-

Informed:
-

### Context

Users who want to delete their account or transfer billing responsibilities for a project are not currently able to do so.

### Goals

* Allow one user to transfer a project's ownership from themselves to another user
  - support transferring project billing responsibilities
  - support deleting an account while preserving project data

### Approach / Design

#### Prerequisites for allowing project transfer

These items must all be true for a project transfer to be processed:
* There are no unpaid invoices on either the current or new owners' accounts
  - if current owner has unpaid invoices, it might mean there is usage for this project that we have not been paid for
  - if new owner has unpaid invoices, it might mean the project is being transferred to a user who won't pay for usage
* new owner is paid tier (it is okay if current owner is free tier)
* new owner has not hit their project count limit
* current owner verifies via email that they approve the transfer of ownership
* new owner verifies via email that they approve the transfer of ownership, including billing responsibilities for the project
* neither user is frozen
* there is not a project transfer request in-progress for this project

#### Results of transfer:

* `project.owner_id` changes to new owner's user ID
* project member record for new owner is created or updated with member type "admin"
* old owner remains as project member with type "admin"

#### Satellite UI

The project owner will navigate to the "project settings" page in the satellite UI, and interact with a "transfer project" card (TODO designs)

UI flow (current owner):

* click "transfer project"
  - if there is already a transfer request in-progress, the current owner or new owner associated with that transfer request must cancel/decline it before this option is available
* if any of the current owner prerequisites fail (unpaid invoices or user is frozen), display a message to the user informing them about what they need to do to transfer a project
* if prerequisites for current owner pass, dialog is displayed asking for the email of the account which should become the new project owner - click "continue"
* send current owner a 6-digit code via email, which they input into the dialog in the UI - click "continue"
* display success message informing the current owner that an email has been sent to the new owner with instructions for completing the transfer

UI flow (new owner):

* on all projects dashboard, the project being transferred appears with some action like "complete transfer" (needs design)
* if any of the new owner prerequisites fail (unpaid invoices, project limit, free tier, or user is frozen), display a message to the user informing them about what they need to do to complete the transfer
* if new owner prerequisites pass, dialog is displayed informing the user that by completing the transfer, they will be responsible for paying for usage on the project - click "confirm/continue"
* send new owner a 6-digit code via email (a different code than was sent to the old owner), which they input into the dialog in the UI - click "continue"
* display success message informing the new owner that the project has been successfully transferred. Send confirmation emails to both the transferer (old owner) and transferee (new owner).


#### Emails

* Current owner verification: Email informing current project owner that they requested to transfer `project name` to `new owner email`, with 6-digit confirmation code.
* New owner verification: Email informing new project owner that they are accepting `project name` from `old owner email`, with 6-digit confirmation code.
* Confirmation email: Sent to both participants, indicating that `project name` has been transferred from `old owner email` to `new owner email`.

#### DB updates

The things that need to be tracked in the DB:

* project ID to transfer
* user ID of transferer (old owner)
* user ID of transferee (new owner)
* verification code for transferer (old owner)
* verification code for transferee (new owner)

We should create a new table for this, e.g. `project_transfer_requests`.

## Disclaimers

### Anti-goals

* Do not allow transferring projects in a way that makes it easy/possible to avoid paying for that project's usage
* Do not expose potentially-private information about other users via this feature (e.g. a project owner should not be able to determine that a random user has unpaid invoices via this feature)

### Alternatives considered

### Open question

* after transfer, what project role should the original owner have? "admin" or "member"?

## Reminders

### Security / Privacy

### Observability

### Test plan

### Rollout

### Rollback

## Out of scope
