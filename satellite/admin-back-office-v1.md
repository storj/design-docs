# Admin Back Office v1

Considerations when reading this document.
- We use the terms "web interface" and "API" for referring to each part of the system and we use
  "back office" when we want to refer to the system as a whole.
- We use the term "operator" to refer to people who has access to the back office.

## Essentials

### Header

Date: 2023-07-27
Owner: Ivan Fraixedes
Accountable: TBD
Consulted:
- Product. Reference person [Andi Steinke](https://github.com/stoweandi)
- Customer support team. Reference person [Helene Unland](https://github.com/heunland).
- Production owners.
- Team integrations console squad: [Maximillian von Briesen (Moby)](https://github.com/mobyvb).
Informed:
- QA team.

### Context

Satellite currently has an [administration
API](https://github.com/storj/storj/blob/main/satellite/admin/README.md)
and minimalist [Web interface](https://github.com/storj/storj/tree/main/satellite/admin/ui) which
allows to interact with its endpoints through Web forms.

API authentication is conducted by two mechanism:
- [An unique token secret](https://github.com/storj/storj/blob/main/satellite/admin/server.go#L215).
  The service isn't exposed publicly, the only way to access to it is forwarding the port of the
  corresponding Kubernetes pod.
- [Allowed groups](https://github.com/storj/storj/blob/main/satellite/admin/server.go#L231).
  This requires that the server is accessed through an authorization proxy (e.g.
  [Oauth2 proxy](https://oauth2-proxy.github.io/)):
  - AP1: https://ap1.admin.storj.tools
  - EU1: https://eu1.admin.storj.tools
  - US1: https://us1.admin.storj.tools

Token secret authentication allows to access al the API endpoints. Allowed groups authentication
only to:
- Read user's information.
- Read/Update user's limits.
- Freeze/Unfreeze user's account.
- Unwarn user.
- Read/Update project's limits.

This current web interface is a visualization of the API. An API isn't an interface for
non-technical people, as a result, only production owners use the interface or directly the API.

### Goals

We want to have an intuitive Web interface that can be used by people with different backgrounds
(e.g. Customer support, finance, sales) and becomes it the administration back office of the
satellite for managing users' accounts.

Back office has a fixed set of access levels. Every access level is associated with a list of
groups. Users are authorized to specific access level depending to the groups they belong.

Back office keeps a modification history of all performed operations that altered the data (create,
update,and delete).

Back office can operates without relying in any resource managed by Storj. Any satellite operator
with or without any relation with Storj will be able to use it.

### Approach / Design

#### Requirements

##### Operations

1. Account:
  1. View information: ID, email, full name, paid/free tier, suspended?, user agents, Multi-factor
     authentication enabled/disabled, data placement restriction (geo fence), list of projects,
     limits (download, storage, number of segments, number of projects allowed).
  2. Change email address.
  3. Disable Multi-factor authentication.
  5. Set limits (download, storage, number of segments, number of projects allowed).
  6. Suspend "disable upload/download".
  7. Re-activate  "re-enable upload/download".
  8. Set user agent.
  9. Set data placement restriction (geo fence).
  10. Remove data placement restriction (geo fence).
  11. Delete account:
    1. When the account is clean: no API keys, no data in any of the projects, no unpaid invoices).
    2. When the account isn't clean (delinquent accounts). It include to deletes all the data
       associated to the account.
2. Project
  1. View information: ID, name, creation time, limits (download, storage), user agent, data
     placement restriction (geo fence),  stats (total used storage, total used download bandwidth,
     total segments).
  2. Set user agent.
  3. Send invitation to grant access to another account.
  4. Set data placement restriction (geo fence).
  5. Remove data placement restriction (geo fence).
3. Bucket
  1. View information: ID, name, user agent, data placement restriction (geo fence).
  2. Set user agent.
  3. Set data placement restriction (geo fence).
  4. Remove data placement restriction (geo fence).

##### Job stories

__When a user wants to access__

I want an external authentication proxy to present the ways how they can authenticate.

Acceptance criteria:
- The proxy has to match the users to one or more groups which are associated to roles in the back
  office configuration.
- The proxy has to offer a mechanism for each following request by this same authenticated user is
  passed to the back office without requiring their credentials again.
- The proxy is responsible of managing the credentials (e.g. expiring them, invalidating them,
  etc.).

__When an operator access the web interface__

I want them to be able to select which satellite they want to access.

Acceptance criteria:
- Show a list of satellites that they can access.
- Simple click to one of the satellites in the list to access to its back office.

__When an operator access to the satellite web interface__

1. I want a view of all accounts.

   Acceptance criteria: Account/User visibility:
   - Viewable in a table format.
   - Attributes viewable:
     - User ID.
     - Email.
     - Full name.
     - Number of projects.
     - Created at.
     - Bandwidth limits.
     - Storage limits.
     - User agents.
2. I want them to be able to search for the account they want to view or make changes.

   Acceptance criteria:
   - Users can be searchable by: user ID, user full name, user email and project ID.
3. I want them to be able to select the account within the search results, so that they are taken to
   an account details view.

   Acceptance criteria:
   - The user ID is clickable and navigates the user to its detailed view.
   - The operator is able to navigate back to the all Accounts table view.

__When an operator navigates to the account details view__

1. I want the data in the account details view to only appear editable if they have access to
   edit/modify, so that it’s clear what they are able to select and it’s clear what they are making
   changes to.

   Acceptance criteria:
   - Identifiable fields that are editable by the operator.
2. I want a confirmation message or visual to display once the change has been made so that so that
   the user knows the change went through.

   Acceptance criteria:
   - Success message if the change succeeded.
   - Error message if the change failed.
3. I want a history of this account's changes displayed at the bottom of the account details page so
   that we can see the previous changes associated with selected account.

   Acceptance criteria:
   - Table view of:
     - Timestamp: Date/Time of operation.
     - Operation: what operation changed.
     - Project: projectID where changes occurred.
     - Bucket: bucket name where changes occurred.
     - Updated: Updated state of operation variables.
     - Last: Last state of operation variables.
     - Operator: Email of who conducted the change.
4. I want there to be a simple way back to the accounts table view so that they can quickly get the
   information needed.

__When changes need to happen to a project__

I want to be able to view project details per account and by project ID so that we can view and
select the details of the project we want to change.

Acceptance criteria:
- We can see the project names and IDs.
- We can apply user agents to projects.
- We can see if a geo fence is applied and what the name of that geo fence is.
- We can see the project limits and project usage.
- Authorized users can select and change:
  - The geo fence if the project is empty.
  - The limits of the project: egress, storage, segments, buckets.
  - Delete the project.

__When changes need to happen to a bucket__

I want to be able to view bucket details per account/per project so that we can modify/add user
agents to buckets and apply geo fence settings to buckets.

Acceptance criteria:
- We can see the bucket names per project ID.
- We can apply user agents to buckets.
- We can see and apply a geo fence to an empty bucket.
- We can see usage per bucket.

__When geo fence needs to be applied to an account__

I want authorized users to select a geo fence setting on the account details view and select the geo
fence region to be applied to new projects/buckets.

  Acceptance criteria:
  - Drop down selection of available geo fence regions.
  - The existing project and buckets will NOT get the geo fence added.

__When an account needs to be suspended__

1. I want authorized users to see a simple "suspend" button in the user's detail view with a drop
   down to select the suspension reason so that we distinguish the different suspension events (e.g.
   "billing account freeze", "other account freeze").

   Acceptance criteria:
   - Button shrinks the user's limits to 0, so that they can no longer upload or download.
   - Drop down to select the reason for suspension:
     - a) account delinquent.
     - b) illegal content.
     - c) Malicious Links.
     - d) other.
   - Unauthorized users should not see this button or drop down.
2. I want the account details to show that it's obvious if a user is suspended so that they can
   easily be manually unsuspended if needed.

   Acceptance criteria:
   - The UX makes it quick to unsuspend a user once suspended

__When an account needs to be unsuspended/re-activated__

I want authorized users to see a simple "reactivate" button in the user's detail view.

Acceptance criteria:
- Button resumes a user's limits to what they had them set at before suspension occurred.
- Add an optional admin note of the reason for unsuspending.

__When an account needs a limit increase or decrease__

I want authorized users to be able to input the allotted usage for:
- Storage limit, and select GB or TB.
- Egress limit, and select GB or TB
- Segment limit, and enter the max number with formatted numerical input fields for better
  readability.
- Bucket limit, and enter the number of buckets.

__When an account needs to be deleted__

I want authorized users to be able to press a "delete account" button along with a "confirm" delete
button that confirms this user and all their information and stored data are to be deleted, so that
we don't have to manually access their API keys and manually delete their data.

__When an account needs to be created__

I want authorized users to have an option in their initial login view to create a user.

This operation rarely happens so the design of the UX shouldn’t make it prominent

Acceptance criteria:
- Input fields for: full name, email, generate (temporary) password.

__When an account needs more projects__

I want to locate the user I need to enable a higher project allowance for and have an option to
increase the number of projects the account is allowed to have in the account details view.

Acceptance criteria:
- Field to increase or decrease the account's project count.
- Project limits default to the user's project limits or can be set if otherwise.

__When a user needs to be added to a different user's project__

I want to be able to search for the project ID.
I need to add the user to, so that I can view the members off that project and add and/or remove a
user.

__When a user needs their multi factor authentication (MFA) reset__

I want there to be a MFA reset option in the account details view.

Acceptance criteria:
- We can see that MFA is enabled or disabled.
- We can only disable MFA.

__When a user wants to change the email address tied to their account__

I want to be able to edit the email address in the account details view so that all events and data
associated to that user is now under the updated email address.

##### Modifications history

The back office keeps a register of all performed operations and who executed them that cause
modifications.

Each record has the following information:
- Timestamp. When the operation was performed.
- Operator email. Who performed the operation.
- Account ID. What account was modified.
- Entity that was modified (i.e. account, project, bucket) and ID.
- Operation. What was done (e.g. change email address, set user agent, etc.). The text used to
  identify the operation has to be unique inside of the entity and human friendly.
- Current data. The new data that was modified with this operation. This is a structured blob
  because it could be more than one modified field per operation.
- Previous data. The existing data previous to the modification performed by this operation. This is
  a structured blob because it could be more than one modified field per operation.

This information is stored in the same database that the satellite uses.

The _DBX_ schema is

```
model admin_operation_modifications_history (
    key id

    index (
        name account_id_index
        fields account_id
    )

    // id is a unique identifier for the operation.
    id                      blob
    // performed_at indicates when the operation was performed.
    field performed_at      timestamp (autoinsert)
    // operator_email indicates which authorized user performed the operation.
    field operator_email    text
    // account_id indicates on what customer the operation was performed.
    field account_id        blob
    // entity_name indicates the entity involved in the operation.
    field entity_name       text
    // entity_id indicates the ID of the entity involved in the operation.
    field entity_id         blob
    // current_data is the values that are created or updated by the operation.
    // This is a structured blob defined by the application and each operation
    // has its structure.
    // It's null for delete operations.
    field current_data      blob (nullable)
    // previous_data is the values that were deleted or were set before were
    // updated.
    // This is a structured blog defined by the application and each operation
    // has its structure.
    // It's null for create operations.
    field previous_data     blob (nullable)
    // caused_by is ID of the operation that causes this one.
    // An operation is cased by another one because the higher one implies it.
    // For example delete and account implies to delete all its projects, so
    // if a project is deleted because of an account deletion, the delete
    // project operation reference the account deletion operation.
    field caused_by         admin_operation_modifications_history.id (nullable, updatable)
)

create admin_operation_modifications_history ( )

read paged (
    select admin_operation_modifications_history.performed_at admin_operation_modifications_history.operator_email admin_operation_modifications_history.account_id admin_operation_modifications_history.entity_name admin_operation_modifications_history.entity_id admin_operation_modifications_history.current_data admin_operation_modifications_history.previous_data
    where admin_operation_modifications_history.account_id = ?
    orderby desc admin_operation_modifications_history.performed_at
    suffix ReadLastOperationsOnAccount
)

read paged (
    select admin_operation_modifications_history.performed_at admin_operation_modifications_history.operator_email admin_operation_modifications_history.account_id admin_operation_modifications_history.entity_name admin_operation_modifications_history.entity_id admin_operation_modifications_history.current_data admin_operation_modifications_history.previous_data
    where admin_operation_modifications_history.account_id = ?
    where  admin_operation_modifications_history.entity_name = ?
    orderby desc admin_operation_modifications_history.performed_at
    suffix ReadLastOperationsOnAccountAndEntityName
)

read paged (
    select admin_operation_modifications_history.performed_at admin_operation_modifications_history.operator_email admin_operation_modifications_history.account_id admin_operation_modifications_history.entity_name admin_operation_modifications_history.entity_id admin_operation_modifications_history.current_data admin_operation_modifications_history.previous_data
    where admin_operation_modifications_history.account_id = ?
    where  admin_operation_modifications_history.entity_id = ?
    orderby desc admin_operation_modifications_history.performed_at
    suffix ReadLastOperationsOnAccountAndEntityID
)

update admin_operation_modifications_history (
  where admin_operation_modifications_history.id = ?
)
```

The _DBX_ schema requires the application to define the data structure for the following fields of
the `admin_operation_modifications_history` model:
- `current_dat`
- `previous_data`

The fields will have different structure depending of the operation, but both will have the same
structure when the performed operations is an entity update and when the operations is:
- An entity creation: `previous_data` field is `null` because there weren't previous data.
- An entity deletion: `current_data` field is `null` because there is not new data.

Each operation type have a structure that depends on its involved data.

The structured data will be stored in JSON format because it's simpler to unmarshall it for sending
it through the API to the clients.

The update operation is needed because when an operation is caused by another one, we have to
perform first the ones they are caused, then the one that causes the others, and finally update the
ones caused with the ID of the one that caused them.

##### Authentication / Authorization

The back office relies on an external authentication proxy. The authentication proxy must send the
groups which the authenticated user belongs.

The back office has the following roles:
- administrator: allowed to perform any operation.
- viewer:
  - Account:
    - View information.
  - Project:
    - View information.
  - Bucket:
    - view information.
- finance manager. All the permissions that the _viewer_ role has and:
  - Account:
    - Suspend.
    - Re-activate.
    - Delete account.
- customer support. All the permissions that the _viewer_ role has, and:
  - Account: All the operations except:
    - Delete account: When the account isn't clean.
  - Project: All the operations
  - Bucket: All the operations.


| Entity   | Permission             | Viewer  | Customer Support   | Finance Manager   |  Admin  |
|:---------|:-----------------------|:-------:|:------------------:|:-----------------:|:-------:|
| Account  | View                   | ✅      | ✅                 | ✅                | ✅
|          | Change Email           | 🚫      | ✅                 | 🚫                | ✅
|          | Disable MFA            | 🚫      | ✅                 | 🚫                | ✅
|          | Set Limits             | 🚫      | ✅                 | 🚫                | ✅
|          | Set Data Placement     | 🚫      | ✅                 | 🚫                | ✅
|          | Remove Data Placement  | 🚫      | ✅                 | 🚫                | ✅
|          | Set User Agent         | 🚫      | ✅                 | 🚫                | ✅
|          | Suspend                | 🚫      | ✅                 | ✅                | ✅
|          | Re-activate            | 🚫      | ✅                 | ✅                | ✅
|          | Delete (clean)         | 🚫      | ✅                 | ✅                | ✅
|          | Delete (no clean)      | 🚫      | 🚫                 | ✅                | ✅
| Project  | View                   | ✅      | ✅                 | ✅                | ✅
|          | Set Limits             | 🚫      | ✅                 | 🚫                | ✅
|          | Set Data Placement     | 🚫      | ✅                 | 🚫                | ✅
|          | Remove Data Placement  | 🚫      | ✅                 | 🚫                | ✅
|          | Set User Agent         | 🚫      | ✅                 | 🚫                | ✅
|          | Send Invitation        | 🚫      | ✅                 | 🚫                | ✅
| Bucket   | View                   | ✅      | ✅                 | ✅                | ✅
|          | Set Data Placement     | 🚫      | ✅                 | 🚫                | ✅
|          | Remove Data Placement  | 🚫      | ✅                 | 🚫                | ✅
|          | Set User Agent         | 🚫      | ✅                 | 🚫                | ✅

The back office configuration allows to specify the relation between roles and groups.

If users belongs to several groups then they will be authorized with the group with the less
restrictive role.

When the different satellites are hosted in different isolated networks having a single
authentication proxy requires to establish links between the networks, so the proxy can access to
all the services. A solution for not having to peer the networks is to have an authentication proxy
for every satellite, which also mitigates the single point of failure of a single authentication
proxy.

When using an authentication proxy for each service we have to configure them in a way that users
don't have to log in more than once when they try to access to a different satellite that the one
that they logged in for satisfying the _users stories_.

For Storj DCS, we determined that the we want to authenticate the users through Oauth2 with their
Storj email account, which is hosted by Google Workspace. We are going to authorize assigning them
to different groups and mapping those groups to the back office roles.

For authenticating the users we will use one [Oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/)
for each satellite and for the "service" that will show the list of satellites, henceforth "selector
service".

All the Oauth2-proxies must use the same Oauth2 client credentials for not requiring the users to
log in when they access to a different satellite or from the "selector service" once they logged in
successfully in one of them and all the satellites and the "selector service" must be hosted under
the same base domain or sub-domain. You can see [this proof of concept](https://github.com/ifraixedes/poc-oauth2-proxies-single-auth)
for how to achieve it.

##### Implementation

We are going to use our current [API generator](https://github.com/storj/storj/tree/main/private/apigen)
for implementing the back office.

This API generator takes a API definition (e.g.
https://github.com/storj/storj/blob/main/satellite/console/consoleweb/consoleapi/gen/main.go) and
generates:
* The server part (e.g. https://github.com/storj/storj/blob/main/satellite/console/consoleweb/consoleapi/api.gen.go).
* The client part (e.g. https://github.com/storj/storj/blob/main/web/satellite/src/api/v0.gen.ts)
* The documentation (e.g. https://github.com/storj/storj/blob/main/satellite/console/consoleweb/consoleapi/apidocs.gen.md)

NOTE we may need to improve the API generator and add more functionalities to support every feature
that we need to implement the back office API.

###### Back-end

The API has to perform the operations based on the user roles specified in the
_Authentication / Authorization_ section and be able to configure the groups associated to them.

The API has to use the logging and metrics systems that we use for the rest of the satellite and
using the similarly.

Because the operations that the users can perform with the back office, the API MUST log all the
operations performed by the users using a specific "log name" to ease filtering only them. We are
using the
[same mechanism for logging all the requests to the satellite console](https://github.com/storj/storj/blob/main/satellite/console/consoleweb/server.go#L1170).
Every log must include the API call and the operator's email who has executed it, but it doesn't
have to contain any data from the request related to the customer, including URL paths that are
parametrized with users'd data (e.g. email address) for not leaking any personal data into the logs.

We'll delete the current admin API once this new one is rolled on.

###### Web UI

We have to build a new Web UI from scratch with the Vue framework and the Vuetify component
framework. The Web UI is already in development and it almost have every component and page that the
back office requires, you can see it on https://storj.github.io/admin-ui/login.

We need to add the API calls and UI logic (client side validations, operation confirmation, etc.)
and all the Javascript client side code that those require.

The current admin web UI will be deleted once this new one is rolled on.

## Reminders

### Security

Everything related to uses' authentication relies on the external authentication proxy. A shallow
list of proxy responsibilities are:
- Users account management.
- Users' sessions management.
- Security on the authentication method.

Because of the operations that the back office allows, we have to pay special attention and take all
the measures that we can for protecting the site from Cross-Site Request Forgery (a.k.a. CSRF).

In the case of using cookies by the back office or the external proxy that are used for
authentication/authorization, we must consider to use as many security measures as we can:
- [Restrict their access](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#restrict_access_to_cookies).
  We should always set the `Secure` and `httpOnly` attributes.
- [Domain attribute](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#domain_attribute). We
  should endeavor for leaving it blank.
- [SameSiteattribute](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#samesite_attribute).
  We should endeavor to use the `Strict` value, however, we already know that we cannot use `Strict`
  the
  [Oauth2-proxy cookies](https://github.com/ifraixedes/poc-oauth2-proxies-single-auth#relevant-findings).
- [Cookie prefixes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#cookie_prefixes). We
  should endeavor to use the `__Host-` prefix, however, we already know that with the Oauth2-proxy
  cookie we can only use the `__Secure-` prefix because we need to set the domain in that cookie to
  be able to send it across the different satellites and the "selector service".

All the web resources (stylesheets, fonts, etc) must be served from the same back office, so we must
not use any external URL.

We should plan to spend some time looking at the
[OWASP cheatsheet series](https://cheatsheetseries.owasp.org/index.html)
for discovering how we could harden more these back office due to the sensitive operations that can
be conducted.

### Observability

We plan to use the same tools that we currently use for observability for the satellites.

### Test Plan

We write integration tests for each API endpoint, including successful, error responses due to
invalid requests, unauthenticated, and authorized requests.

We coordinate with our QA team to establish a test plan for the Web UI.

### Rollout

We plan to use the same rollout procedure that we are having with the current admin UI/API.

## Rollback

We don't plan to have any rollback strategy because only people who works or is hired by the
organization that runs the satellite will use the service, so we believe that we can fix the bugs
without an special urgency and, in the case that whole service is down, fixing the issues with a few
hours of downtime should be fine.

Nonetheless, we must always consider that new versions don't make changes in the database schema
that restoring a previous version wouldn't be possible because of them. We establish that new
versions must keep the data schema compatible with the last 2 older versions, hence, the third
should perform the data schema changes that weren't done to keep the compatibility with the previous
ones. We establish that two deployed versions should be enough to detect flaws and issues that they
force us to restore the service to run a previous version.

## Out of scope

### User features

- Account:
  - Billing information.
  - Stripe information.
  - Stats: number of projects, total used storage, total used download bandwidth, total segments.

### System features

#### Avoid undesired request by a stale UI

A stale UI happens when a user had loaded the Web UI before a new version is rolled out and it uses
the UI when the new version is rolled out without refreshing the browser.

This may provoke some undesired data updates without the user knowing it. For example, a new version
has added a new data field to one of the operations that accepts to se ti to null; the previous UI
and API client will continue sending a valid request without sending that field, but because the new
API accepts null will unmarhsal the missing field to null overriding the value.

For avoiding stale UIs, the API an always expect a specific header with a unique value that will
change on every new version (e.g. git commit hash); the API rejects a request that doesn't send the
value or an value that doesn't match with a specific error that the client can differentiate from
other errors, so the UI can inform the user about it and refresh the browser to update the UI to the
current version.

If the UI is served by an independent server, that server MUST have a configuration field to
indicate the unique value corresponding to the back office version whose UI is serving.
