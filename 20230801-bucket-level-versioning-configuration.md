tags: [ "satellite", "object versioning", "bucket level versioning" ]
---

# Bucket level versioning configuration

## Abstract

This is a design document and implementation strategy for maintaining bucket level configuration metadata used to 
manage versioning for objects within the bucket.

## Header

Date: 2023-08-01<br>
Owner: Damein Morgan<br>
Accountable: 
- Team Delivery squad ([Jen](https://github.com/jenlij))
- Team Satellite squad ([jt](https://github.com/jtolio))

Consulted:
- Production owners. 
- Team satellite.

Informed:
- Team integrations edge.

## Background/context

While the focus of this doc is on maintaining bucket level versioning configuration metadata, it is important to 
understand [how object versioning is handled in the system](https://docs.google.com/document/d/1AMWDRncVsCUXBp1Sqk6uHibcM6omhdRwtNtJfr3cpDc/edit?pli=1#heading=h.a5g1c8b5x9f1). 

## Goals
Some high level design goals include:
* Full S3 compatability
* Extensibility 
* Customizable defaults
* Minimal impact

### Full S3 compatability
One of the primary goals of this design is compatability with the [S3 API for enabling and/or disabling 
versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/manage-versioning-examples.html) at the bucket 
level. While our implementation details may differ, standard s3 api calls should behave as expected.

### Extensibility
The design should support or not deter additional bucket level configuration and/or constraints in the metadata.

### Customizable defaults
Support for customizable defaults that can be defined and configured at the project/account level.

### Minimal impact
The operational impact to the system with the addition of this bucket level configuration should be limited to the
API calls of version based operations only.

These API calls include
* `GET Bucket versioning`
* `PUT Bucket versioning`
* `GET Object`
* `PUT Object`
* `DELETE Object`
* `List Object Versions`
* `Copy Object`
* `Upload Part Copy`
* `Restore Object`


## Design and implementation
In AWS S3, Versioning is a per-bucket subresource, with 3 possible states. By default, newly created buckets are in 
the `unversioned` state. Once versioning for a bucket is `enabled` its state can only be changed to `suspended` or 
re-`enabled`.

To support AWS s3 compatability, we will add a `versioning_state` field to the `bucket_metainfo` table. The state of 
this field will be consulted during certain operations to allow for the correct versioning behavior. Unlike the AWS 
versioning subresource, the `versioning_state` field will have 4 possible states, `unversioned`, `enabled`, 
`suspended`, and a new state called `unsupported`. The first three states all behave in the same matter as the AWS 
states. The new `unsupported` state will be used to indicate that the bucket does not support versioning, and will 
be the state for all buckets that exist prior to the completion of the object versioning implementation.

To support customizable defaults, we will add a `default_versioning_state` field to the `project` table. The state of
this field will be consulted during bucket creation to set the initial state of the `versioning_state` field for the
bucket. The default value for this field will be `unversioned`.

### Database Schema
#### Bucket Metainfo
The `versioning_state` field will be added to the `bucket_metainfo` table.

```
model bucket_metainfo (
    ...
    field versioning_state    int (updatable)
)
```

This int type will have 4 possible values: 0 (`unsupported`) which represents a bucket for which versioning is not 
supported. Any bucket with a versioning state of `unsupported` cannot transition to any other versioning state. A 
value of 1 (`unversioned`), represents a bucket which is eligible for object versioning, but has never had it 
enabled. A value of 2 (`enabled`) represents a bucket which has versioning enabled, and a value of 3 (`suspended`) 
represents a bucket which has versioning suspended. By default, new buckets will have a versioning state of 
1 (`unversioned`), however, the default value can be overridden by the `default_versioning_state` field in the 
`project` table.

| Value | Description          |
|-------|----------------------|
| 0     | unsupported          |
| 1     | unversioned          |
| 2     | enabled              |
| 3     | suspended            |

Valid transitions are:

 - `unversioned` -> `enabled` (enable versioning)
 - `enabled` -> `suspended` (suspend versioning)
 - `suspended` -> `enabled` (unsuspend versioning)

Given that the only valid transitions are to states `suspended` or `enabled`, the public API for versions only needs 
to support these values. The `unsupported` and `unversioned` states will be handled internally by the system.

#### Project
The `default_versioning_state` field will be added to the `project` table.

```
model project (
    ...
    field default_versioning_state    int ( updatable )
)
```

This int type will have one of two states: 0 (`unversioned`) or 1 (`enabled`). By default, new projects will have a
default versioning state of 0 (`unversioned`).

| Value  | Description          |
|--------|----------------------|
| 0      | unversioned          |
| 1      | enabled              |

### Alternatives considered
During discussion of the design, it was considered to allow for only 3 states to exactly match the AWS specification.
This however would require a migration of all existing buckets to a new table in ordered to support 
versioning for pre-existing buckets. This would have been a significant effort, and was determined to be unnecessary.

During discussion of the design, it was also considered to allow for only 2 states of the AWS versioning subresource.
This would have been achieved by an optimization that combined removing the ability for a user to suspend object 
versioning of a bucket, while also defaulting all newly created buckets to `disabled`. This would have allowed for a 
simpler implementation, but wouldn't allow for the design goal of full compatability with the AWS API.

## Out of scope
Gateway ST implementation of SetBucketVersioning, GetBucketVersioning and ListObjectVersions should be performed on 
the **_gateway_** layer. Care will need to be taken to honor both the marker and versionMarker so that pagination works 
properly with both the object and object versions.

Gateway MT implementation and forward of SetBucketVersioning, GetBucketVersioning, and ListObjectVersions should be 
performed on the **_project_** layer. The default handlers in minio for SetBucketVersioning and GetBucketVersioning 
are hard coded to return no support for bucket versioning when operating in "gateway" mode (which is what we use). 
Even when operating in non-"gateway" mode, the versioning state is stored in a special object in the bucket, which 
isn't compatible with the bucket metainfo approach the Satellite will implement getting/setting the bucket 
versioning state.

In addition to managing object versions by suspending and re-enabling the functionality at the bucket level, users 
must also be provided a robust method for managing the lifecycle of objects in a version enabled bucket.