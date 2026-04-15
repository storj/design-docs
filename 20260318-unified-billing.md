---
tags:
  - satellite
  - billing
  - invoicing
version: 0.0.1
---

# Unified Billing

## Essentials

Storj tracks usage to invoice their customers across the multiple production satellites separately
without offering a unified view, invoice, and payment to them per billing cycle.

### Header

Date:  2026-03-18

Owner: Ivan Fraixedes (@ifraixedes)

Accountable:
- Console team

Consulted:
- @mobyvb

Informed:
- @NiaStorj

### Context

#### Technical details
Storj operates 4 production satellites in different regions:
- Asia (AP1)
- Europe (EU1)
- United Kingdom (UK1)
- USA (US1)

Each satellite has its own isolated database (currently a different Spanner instance) and track the
user accounts' usage in the database.

Each satellite track usage for several purposes; this document refers to usage tracking only for
customer billing purpose.

The satellite have some command-line commands to aggregate the customer usage individually for a
specific month, prepare invoices, send  invoices, and emit payments via
[Stripe](https://stripe.com). We execute these satellite commands manually in Kubernetes Batch Jobs
using Helm charts.

Each satellite uses a different Stripe account.

Customers that have one account in 2 or more satellites get an invoice and the corresponding usage
charge for each satellite account.

Previous to this design document some changes were applied to send all the usage tracking data to
our BigQuery data warehouse through [Eventkit](https://pkg.go.dev/storj.io/Eventkit) aside of
tracking it in the Satellite database. Commits:
* https://review.dev.storj.tools/c/storj/storj/+/19739 --> the main commit
* https://review.dev.storj.tools/c/storj/storj/+/20471
* https://review.dev.storj.tools/c/storj/storj/+/19861
* https://review.dev.storj.tools/c/storj/storj/+/19765
* https://review.dev.storj.tools/c/storj/storj/+/20468
* https://review.dev.storj.tools/c/storj/storj/+/20495
* https://review.dev.storj.tools/c/storj/storj/+/20498
* https://review.dev.storj.tools/c/storj/storj/+/20499

##### Evenkit considerations

Eventkit was conceived for telemetry, specifically to report multidimensional events. Initially was
sending data over UDP, but because it has an abstraction how the data is sent to destinations, it
isn't bound to any protocol.

In the satellite is configured using the [`storj.io/Eventkit/bigquery`](https://pkg.go.dev/storj.io/Eventkit/bigquery)
package in [`storj/shared/modular/Eventkit`](https://github.com/storj/storj/blob/main/shared/modular/Eventkit/Eventkit.go).

In all the productions satellites, we have configured a BigQuery destination using parallel and
batch. All of them are:
```
bigquery:appName=<satellite-name>,project=storj-data-science-249814,dataset=Eventkitd3|parallel:workers=10|batch:queueSize=20000,batchSize=4000,flushInterval=15s`
```

These is passed to the [`bigquery.CreateDestination`](https://pkg.go.dev/storj.io/Eventkit@v0.0.0-20250410172343-61f26d3de156/bigquery#CreateDestination)

A destination have a `Submit` method that it's called by `Scope.Event` with the passed events.

Each destination operates as follow when its `Submit` event is called:
* BigQuery: It uses the name and tags to determine the BigQuery table and insert into it the
  _application_ received when it's created, the source (the host name if it's available), the
  current timestamp, the event's timestamp, and the event's tags.
* Parallel: It sends the events to a buffered channel of the same amount of workers (i.e.
  `gororutines`, 10 per the above configuration example). Each worker calls `Submit` to the
  associated destination on the `Run` method which doesn't exit until context is done. Each worker
  has an instance of the same destination created with the passed _destination "constructor"_.
* Batch: It sends the events to a buffered channel (`queueSize` configuration parameter). The `Run`
  method execute and infinite loop that simultaneously:
  * Read events from the channel and append them a list (i.e. `slice`); if the list's length is
    greater or equal than the batch size (`batchSize` configuration parameter) it calls `Submit` to
    the destination provided when instantiated
  * Every flush interval check the list of events and if it's greater than 0, it calls the
    destination `Submit` method passing them.
  * When context is done, it drains the existing events in the buffered channel to add them to the
    list of events and calls the destination `Submit` method passing them.

These 3 destinations are composed together into one by wrapping the previous in the reverse order of
the above list `Batch(Parallel(BigQuery))`.

This architecture is designed to have multidimensional telemetry data, and by definition, telemetry
should have a negligible performance and resource consumption on the application functionalities, so
the event ingestion doesn't guarantee that all the events are recorded.

Events aren't recorded under the following circumstances:
* BigQuery destination error when inserting events or it doesn't succeed in 1 minute
  * The timeout is set by [`Eventkit/bigquery.BigQueryClient.SaveRecord`](https://github.com/storj/Eventkit/blob/61f26d3de156/bigquery/client.go#L114)
  * The same `SaveRecord` method returns an error performing one of its operations which involves
    calling BigQuery API for different purposes
  * Under this circumstance the error is printed out in the [`BigQueryDestination.Submit` method](https://github.com/storj/Eventkit/blob/61f26d3de156/bigquery/client.go#L114)
* Parallel destination will stop sending messages when the `Run` method is stopped via context
  cancellation because the worker will exit, however, new events can be submitted between the
  workers are exiting and before the channel is closed which will be dropped when the channel is
  closed.
* Batch destination drops messages when the [buffered channel is full](https://github.com/storj/Eventkit/blob/61f26d3de156/destination/batch.go#L105).
  It also some messages won't be sent if `Run` is stopped via context cancellation and it isn't
  called again, despite it drains the buffered channel, it does for the its length at certain time,
  and client can continue submitting events.

BigQuery destination barely fails submitting events. The [US1 logs of a 30 days window](https://console.cloud.google.com/logs/query;query=resource.type%3D%22k8s_container%22%0Aresource.labels.project_id%3D%22storj-prod%22%0Aresource.labels.location%3D%22us-east4%22%0Aresource.labels.cluster_name%3D%22us-east4-gke-fern%22%0Aresource.labels.namespace_name%3D%22satellite%22%0AtextPayload%3D~%22WARN:%20Couldn't%20save%20Eventkit%20record%20to%20BQ:%22%0A;storageScope=project;cursorTimestamp=2026-03-20T03:09:20.432652942Z;duration=P30D?project=storj-prod)
executed at the time of writing this sentence showed only 8 errors in total and all of them related
to context deadline exceeded.

Batch destination is more important, in the last 24 at the time of writing this sentence the [US1 had
dropped 30250 events](https://thanos.storj.rodeo/d/ivwnsll/satellite-Eventkit?orgId=1&from=now-24h&to=now&timezone=UTC).

#### Business details

Storj current pricing tiers are:
* Regional: $10/TB. 1x egress per month included, additional egress $0.01/GB
* Archive: $6/TB. Egress $0.02/GB
* Global: $15/TB. 1x egress per month included, additional egress $0.02/GB

Minimum usage is $5/month, unless it's prepaid with STROJ tokens or via a partner.

All our tiers have a 30 days trial period limited to 25 GB.

### Goals

Customers see all their usage spread through several satellites unified in a same web interface and
their only get one charge for the entire usage per billing cycle.

Customers can see the usage of each specific satellite and they receive the usage per satellite in
the invoice details.

### Approach / Design

#### Globally tracking usage

Each satellite has its own separated database, at this time, they use a different Spanner instance
or database. These database aren't interconnected.

Satellites send usage data into our BigQuery through
[Eventkit](https://pkg.go.dev/storj.io/Eventkit).

Eventkit is susceptible to drop events when the satellites are shutting down or the number of
reported events is too high with the ingestion buffer capacity.

These amount of dropped events may impact the total usage tracked in data warehouse and cause a
considered deviation with the one registered in the satellite DB, which may cause a loss in revenue
if we invoice clients based on the data warehouse tracked usage.

We have to build monitoring and alerts when this happens to detect possible important misaligned
usage.

We have to document the procedure to inspect these misalignments and how to align them in case that
they have an important revenue loss impact. Finance should establish a threshold about what's
considered an important revenue loss.

#### Unified view

We currently use Stripe for invoices and payments.

Stripe's Customer Portal feature allows to build a portal where our customers could:
* View and manage their active subscriptions
* Update payment methods
* View billing history and invoices
* Track usage for usage-based subscriptions

It offer two options:
1. No-code solution: Use Stripe's pre-built Customer Portal
   * Fully hosted by Stripe
   * Customizable branding and functionality
   * Minimal integration required
2. Custom solution: Build your own portal using Stripe APIs
   * For more customized experiences
   * Requires more development work

##### Stripe's customer portal

Stripes customer portal is hosted by Stripe.

The clients can access to the portal through:
* A general link where they can login with their registered email and one-time passcode that they
  receive to that email each time that they desire to login
* A secure, temporary link generated with the Stripe's API that redirects the user to the Stripe's
  hosted portal with their specific credentials

The no-code solution only allow to specify a global configuration for all the customers, while with
the API we can change the configuration of the portal programmatically for each customer based on
business logic. The API also offers to create deep links to customize the portal flow, such as:
* Link directly to the page with the specified action for your customer to complete. Navigational
  components to access the rest of the customer portal are hidden so the customer can focus on the
  single action
* Customize the redirect behavior after the customer completes the action—redirect them immediately
  to your own URL, to a hosted confirmation page, or back to the portal homepage
* Personalize the flow with unique options like prefilled promotion codes or custom messages

At the time of writing this document, the customer portal has the following limitations:
* Subscriptions modifications are limited when:
  *  Customer can cancel it in the portal, but can't update it when a subscription uses any of the
     following:
      * Multiple products
      * Usage-based billing
      * Sending invoices for collection
      * Unsupported payment methods
  * Customers can't update or cancel subscriptions that currently have an update scheduled with a
    subscription schedule
  * Customers can only modify subscriptions if the new price has the same tax behavior as the
    initial price. Additionally, no modifications are allowed if the tax behavior is unspecified,
    even if the tax behavior of the new price is unspecified
  * Customer modifications to a trialing subscription end the free trial and create an invoice for
    immediate payment
  * When customers can switch plans, only a maximum of 10 products can be specified for them to
    choose from
* Technical limitations
  * When the payment method management is enabled for the session, the portal displays the payment
    method section, even if the portal doesn't support the customer’s default payment method
  * Not possible to define multiple Prices with the same product and recurring.interval values. This
    can be solved creating different product versions
  * The customer portal cannot be displayed inside an iframe

Because our pricing is per usage we have to use Stripe metered usage with a subscription, however,
we cannot auto invoice the customers every month which have at least one bucket in Regional or
Global tier because they have 1x egress included, which is variable to the storage, and Stripe
doesn't support these kind of business rules. This implies to operates like we do nowadays, which is
create draft invoices, add items regarding to the additional egress, and finalize them to charge our
customers.

The $5/month minimum fee cannot be modeled in Stripe. We can use billing thresholds to avoid auto
invoicing when they have lower usage and implement the logic in our process to adjust them and
charge the clients. However, because we are only limited to use the auto invoicing only for the
Archive tier, it isn't worth to use auto invoice at all, and invoice all the customers from our
system.

Stripe subscriptions allow to set a free trial period with certain price, when it's set to 0, the
subscription will be active, despite the customers won't pay anything on the invoices during that
period.

Stripe metered events have a limit of 2k events/second, hence we have to push events with a retry
backoff, and add monitoring to report if retry gives up after certain attempts.

##### Stripe's customers migration

Each satellite uses a different Stripe account. To unify the billing we have to migrate the clients
of each satellite to the same Stripe account.

Stripe allows to migrate accounts within itself. It has a [list of
considerations](https://docs.stripe.com/get-started/data-migrations/pan-copy-self-serve?copy-method=full#data-considerations).

We need to conserve the customer IDs because we keep them in the satellite DB. Stripe migration
conserves the customer ID, however, we will have to implement some migration tool to unify users who
has an account in more than one production satellite because they currently have a different Stripe
customer ID per satellite.

Our migration tool will have to consider how to attach the clients to the new subscriptions that we
will create if Stripe migration tool cannot do it and we may add other logic if we find that some of
the Stripe's migration considerations don't fulfill our needs.

## Disclaimers

We are aware that using Eventkit may cause usage misalignments because of the message that are
dropped by different causes, however, we are analyzing how significant is before we decide to
migrate to use the global central usage through Eventkit existing instance for the unified billing.

### Anti-goals

* Show unified usage in real-time
* Show unified usage for the current month. Only the usage of passed months and when the invoice
  process is closed will be shown

### Alternatives considered

#### Usage tracking

##### New system

We could build a tracking the usage globally across satellites from the ground up instead of using
Eventkit.

The development of new system will also delay to deliver the feature to our customers and will
require to be exposed in several stages (i.e. Alpha, beta, and release candidate) in order to ensure
that works as expected.

The amount of engineering resources needed and the amount of time to deliver the feature in a stable
version to our customers leaded to discard this alternative.

##### Specific Eventkit instance

We could use an Eventkit instance specifically to track usage to reduce the number of message
dropped due to full buffers.

This requires to add new instance and wire it to all the components that needs to track usage.

This would require to run a new cycle to send the ingested events by this new instance requiring
more satellite compute resources.

Despite of the requirements, the amount of needed engineering and compute resources aren't
considerably big, however, we have already in place to use the same Eventkit instance and unless
that future will show that we have important usage misalignments we discard this solution to save
resources and reduce the time to deliver the feature to the customers.

#### Unified billing

##### Lago: Open source metering and usage based billing API

[Lago](https://getlago.com/) is an [open-source](https://github.com/getlago/lago) payment-agnostic
billing platform designed for complex usage-based models.

Lago offers distinct advantages for teams requiring full control and flexibility:
* No Vendor Lock-in: Lago is agnostic, allowing integration with any payment processor (Stripe,
  Adyen, PayPal, etc.), while Stripe Billing forces exclusive use of Stripe Payments
* Cost Structure: Lago is free open-source software (AGPL v3) where you pay only for
   infrastructure or a fixed platform fee
* Usage-Based Complexity: Lago is built from the ground up for event-based metering, handling
  complex scenarios like yearly plans with monthly overages without pre-aggregating data
* Data Ownership: As a self-hostable solution, Lago allows teams to own their data and run financial
  reports directly on their own Postgres database

However, Lago requires significant engineering investment to build and maintain the billing
infrastructure and customer-facing UI.

Lago's UI offers a self-hosted
* Admin Dashboard: It provides a hosted UI for internal teams to manage billing operations (plans,
  customers, metrics), which is included with the platform
* Customer Portal: It offers a customer portal for end-users to view usage and invoices, the
  open-source version requires you to build your own UI. A pre-built, branded portal is available
  but was historically a premium, paid feature, which recently updates indicate it is now free for
  all users, though it still requires integration effort on your part.

These are the main differences between Lago and Stripe

| Feature | Lago | Stripe Billing |
| :--- | :--- | :--- |
| **License** | Open Source (AGPL v3) | Proprietary |
| **Payment Processors** | Any (Stripe, Adyen, Mollie, etc.) | Stripe Payments Only |
| **Pricing Model** | Free (self-hosted) or Fixed Fee + Volume | 0.5% - 0.8% of revenue + Transaction Fees |
| **Architecture** | Event-based, flexible metering | Subscription-centric, limited usage features |
| **Customization** | Full code access, self-hostable | Limited customization, cloud-only |
| **Engineering Effort** | High (requires assembly/maintenance) | Low (managed, drop-in) |

We discarded Lago because it will require more engineering resources to develop the unified view, in
contrast, Stripe provides a fully managed, feature-rich customer portal as a standard part of its
Billing product, requiring minimal setup.

And, we will also have to continue bound to Stripe for payments because Lago isn't payment
processor, it's designed to work with them.

### Open question

* Should we ingest the messages when Eventkit is tearing down instead of dropping them?

  Reference: https://review.dev.storj.tools/c/storj/Eventkit/+/20817/comment/5e55c7ad_01864c64/
  Note: BatchQueue must be reviewed for this purpose too.
* Should we constantly compare usage tracked in Eventkit with the one tracked per satellite to spot
  relevant misalignments?
* Why do we need to Eventkit and we don't report all the usage from the different satellites
  directly to one single Stripe's account?
* Do we need to build a portal that shows the data instead of redirecting the client to Stripe which
  already show the next usage to bill, the billed usage, the invoices, etc?

## Reminders

### Security / Privacy

Eventkit and Stripe are already used. We have trusted them to receive and store sensitive data and
these changes don't new systems and data with a higher privacy level.

### Observability

We have to verify Eventkit to ensure that it reports through monkit when events are dropped and how
many are dropped.

We have to build a specific dashboard for Eventkit to display the drop event statistics and configure
alerts when reached certain threshold.

### Test plan

Unit tests, integration tests will be conducted on the new implementations when they are not already
covered by the existing tests.

QA team members will conduct the usual test plans as they do with any other new feature.

Once the systems is implemented and deployed, we will run at least one fully billing cycle with the
new system and continue invoicing the client with the current method. When we close the full cycle,
we will take a sample of different customers:
* With high volumes
* With different tiers
* With multiple tiers
* With accounts in more than one satellite
* At least one per satellite

We'll create the invoice draft invoices and will compare them with the ones closed with the current
method. If there isn't a mismatch, we'll identify from the discrepancies are coming and repeat with
a new full cycle if the fixes aren't very trivial.

### Rollout

The implementation will be deployed continuously as we do. We'll add features flags for changes that
cannot be deployed without altering the current system or are visible to our customers.

Once we verify that the new unified billing is working as expected, we will swap the flags and our
customers will use the new unified interface.

The introduced feature flags will be deprecated, changing their defaults to the most usual
production values, and scheduled to be removed after certain period of time that the new unified
billing is in place.

### Rollback

There isn't any specific rollback plan because the system will be introduced incrementally and it
will be enabled once is completely deployed by changing the required features flags once verified
accordingly with the [test plan](#test-plan) and [rollout](#rollout).

In case that we need to use the previous system, we will swap again the feature flags to disable it.

## Out of scope

* Unified usage for any purpose other than billing
* Unified usage limits notifications
* Currently unified usage not billed yet in real-time or at the same cadence than each satellite
  shows to the customer
* Remove the implementation of the previous billing system
* Automate misalignments between satellite registered usage and BigQuery global usage. This document
  requires a documented manual procedure, automating part of the process or entirely without human
  intervention is considered out of the scope because we don't think at this time that this will be
  required frequently.
