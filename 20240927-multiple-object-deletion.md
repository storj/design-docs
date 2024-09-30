tags: [ "gateway", "satellite", "satellite/metainfo", "uplink" ]
---

# Multiple Object Deletion

## Essentials

### Header

Date: 2024-09-24

Owner: Jeremy Wharton

Accountable:
- Edge team

Consulted:
- Product team

Informed:
- QA team

### Context

The satellite metainfo RPC service provides an API through which objects stored on the Storj network may be interacted with. While this service exposes an endpoint through which individual objects may be deleted, it lacks an endpoint for deleting multiple objects in a single request. This limitation impacts services that depend on the metainfo RPC service, such as our S3-compatible gateway, by requiring separate requests for each object in a bulk deletion operation.

This approach has several issues:

- Network overhead: The amount of time spent transmitting and processing network requests increases with the number of objects to be deleted. This not only delays bulk operations but also consumes unnecessary bandwidth.

- Rate limiting: Each deletion request counts against the deletion rate limit of the project associated with the request. This reduces the amount of deletion operations that can be performed within a given time frame. Additionally, this may cause bulk deletions to fail partway if the rate limit is reached mid-operation.

- Database overhead: Each deletion involves the execution of a separate SQL query, increasing the load on the database as the object count grows. Implementing bulk deletions gives us the opportunity to batch queries or delete multiple objects in a single query, reducing database overhead.

For further context, the gateway currently supports Amazon S3's [DeleteObjects](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObjects.html) operation, which allows for the deletion of multiple objects with a single HTTP request. However, the lack of native support for bulk deletions in the metainfo RPC service makes this operation subject to the aforementioned issues, as the gateway must translate every S3 request to a series of RPC requests.

### Goals

The goal of the implementation described by this document is to allow multiple objects to be deleted in a single request to the satellite.

### Approach / Design

#### Satellite

The satellite metainfo RPC service must be extended with an endpoint allowing for the deletion of multiple objects in a single request.

Requests directed to this endpoint must contain the following information:

- The requester's API key
- The bucket from which objects will be deleted
- An indication of whether governance mode Object Lock restrictions should be bypassed
- An indication of whether quiet mode should be enabled.
  - With quiet mode enabled, only errors are returned; information about successful deletions is omitted from responses. Supporting this mode would allow us to save bandwidth on clients that use it.
- A list of objects to delete

The endpoint must respond with the following information:

- A list containing the details of each successful deletion, including:
  - The deleted object's key
  - The deleted object's version (if the version wasn't specified in the request)
  - The version ID of the delete marker (if applicable)
- A list containing the details of each failed deletion, including:
  - An error code
  - The object's key
  - The object's version ID (if it was provided in the request)

#### Libuplink

Libuplink must provide a method through which the satellite metainfo RPC service's multiple object deletion endpoint is contacted. This is necessary for the gateway to function.

#### Gateway

The gateway must be given a configuration option that indicates whether the new multiple object deletion feature should be used. When it is enabled, it must invoke the new Libuplink method.

#### Protobuf

The protobuf definition for the metainfo RPC service must include a method for deleting multiple objects. Additionally, request and response types must be defined.

```proto
service Metainfo {
    rpc DeleteObjects(DeleteObjectsRequest) returns (DeleteObjectsResponse);
}

message DeleteObjectsRequest {
    RequestHeader header = 15;

    bytes bucket = 1;
    bool bypass_governance_retention = 2;
    bool quiet = 3;
    repeated DeleteObjectsRequestItem items = 4;
}

message DeleteObjectsRequestItem {
    bytes encrypted_object_key = 1;
    bytes object_version = 2;
}

message DeleteObjectsResponse {
	repeated DeleteObjectsResponseDeletedItem deleted = 1;
	repeated DeleteObjectsResponseErrorItem errors = 2;
}

message DeleteObjectsResponseDeletedItem {
    bytes encrypted_object_key = 1;
    bytes deleted_object_version = 2;
    bytes delete_marker_version = 3;
}

message DeleteObjectsResponseErrorItem {
    int32 code = 1;
    bytes encrypted_object_key = 2;
    bytes object_version = 3;
}
```

## Disclaimers

### Anti-goals

### Alternatives considered

An alternative to the design described in this document is to implement a separate rate limit specifically for bulk object deletion operations. However, bulk deletion operations could still fail partway through due to hitting the bulk deletion-specific rate limit, even though other operations would remain unaffected. In addition, unnecessary bandwidth consumption would remain an issue, as each object deletion would still require a separate request.

### Open questions

## Reminders

### Security / Privacy

The same security and privacy considerations that apply to the existing single-object deletion implementation must apply to the multiple object deletion implementation. We must ensure that the client is unable to delete any objects for which deletion is not permitted by its API key. Additionally, Object Lock restrictions must be obeyed.

### Observability

We plan to use the same observability practices that we currently use for existing metainfo RPC service endpoints. Each request triggers an Eventkit event containing request metadata, and Monkit instrumentation is added to each non-trivial method involved in the processing of the request.

Customers may enable [bucket logging](https://storj.dev/dcs/buckets/bucket-logging) in order to view access logs for S3 requests - including those for `DeleteObjects` operations - that are sent to our S3-compatible gateway.

### Test plan

Tests for this design should confirm all of the following statements:

- If the bucket is versioned and the version ID of an object isn't specified, a delete marker will be placed at the object's location, and no deletions will occur. This will be the case even if there isn't an object version at the location.
- If the bucket has versioning suspended and the version ID of an object isn't specified, a delete marker will be placed at the object's location. If there's an object version at the location, it will be deleted.
- If the bucket is unversioned and the version ID of an object isn't specified, the object will be deleted without the insertion of a delete marker at its location. If no object exists at the location, an error will be returned.
- If a legal hold has been placed on an object, it must not be deleted under any circumstance.
- If the object has an active retention period, deletion should only succeed if it is locked under governance mode, the requester indicates that governance mode restrictions should be bypassed, and the requester is authorized to bypass governance mode restrictions.
- If the number of objects to delete exceeds 1000, then none of them should be deleted, and an error must be returned.

The following apply only to the gateway:

- The HTTP status code returned by the gateway's `DeleteObjects` endpoint must be `200 OK` unless the request itself fails to be processed. Individual object deletion failures should not affect the status code.

### Rollout

The gateway will be given a configuration flag that enables the use of the new `DeleteObjects` endpoint, which will be disabled by default. This feature must only be enabled for production satellites once QA testing has completed.

### Rollback

Rolling back this feature should only entail disabling its feature flag, causing the gateway to use the old implementation of the `DeleteObjects` operation.

## Out of scope

In addition to the deletion method outlined in this document, which involves specifying a list of object keys, a previous [design document](https://review.dev.storj.io/c/storj/storj/+/11352) concerning this topic described a method for deleting all objects under a specified prefix. However, prefix-based deletion is out of scope for this document.
