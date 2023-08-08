# Object versioning configuration at the bucket level

## Abstract

This is a design document and implementation strategy for maintaining bucket level configuration metadata used to 
manage versioning for objects within the bucket.

## Background/context

While the focus of this doc is on maintaining bucket level versioning configuration metadata, it is important to 
understand [how object versioning is handled in the system](https://github.com/storj/storj/issues/5808). 

## Goals
Some high design goals include:
* Full S3 compatability
* Extensibility 
* Customizable defaults
* Minimal impact

### Full S3 compatability
One of the primary design goals of this doc is compatability with the [S3 API for enabling and/or disabling 
versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/manage-versioning-examples.html) at the bucket level.

### Extensibility
The design should support additional bucket level configuration and/or constraints in the metadata.

### Customizable defaults
Support for customizable defaults that can be defined and configured at project/account level.

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

In AWS S3, Versioning is a per-bucket subresource, with 3 possible states. By default, newwly created buckets are in 
the `unversioned` state. Once versioning for a bucket is `enabled` its state can only be changed to `suspended` or 
re-`enabled`.

To support this, we will add a `versioning_state` field to the `bucket_metainfo` table. The state of this field will be 
consulted whenever operating on an object to enable the correct versioning semantics for that operation. The field 
will be `nullable` and NULL represents "disabled". By default, the field will be NULL like AWS S3, though on bucket 
creation it will be changed to a project/user level default value if specified.

```
model bucket_metainfo (
    ...
    field versioning_state    bool ( nullable, updatable )
)
```

The boolean type can have several states: “true”, “false”, and a third state, “unknown”, which is represented by the 
SQL null value.

| Name    | Storage Size | Description            |
|---------|--------------|------------------------|
| boolean | 1 byte       | state of true or false |

See [PostgreSQL documentation](https://www.postgresql.org/docs/9.1/datatype-boolean.html) for more information.

### Database Schema
Subresources table vs bucket_metainfo table

pros of subresources table:
* more extensible
* easy observations/filtering and analytics
* DB table load is spread out over multiple tables
cons of subresources table:
* more complex
* more expensive to query

pros of bucket_metainfo table:
* less complex
* less expensive to query
cons of bucket_metainfo table:
* less extensible
* harder to observe/filter/analytics

## Out of scope
Gateway ST implementation of SetBucketVersioning, GetBucketVersioning and ListObjectVersions should be performed on 
the **_gateway_** layer. Care will need to be taken to honor both the marker and versionMarker so that pagination works 
properly with both the object and object versions.

Gateway MT implementation and forward of SetBucketVersioning, GetBucketVersioning, and ListObjectVersions should be performed on
the **_project_** layer. The default handlers in minio for SetBucketVersioning and GetBucketVersioning are hard 
coded to return no support for bucket versioning when operating in "gateway" mode (which is what we use). Even when 
operating in non-"gateway" mode, the versioning state is stored in a special object in the bucket, which isn't 
compatible with trhe bucket metainfo approach the Satellite will implement getting/setting the bucket versioning state.