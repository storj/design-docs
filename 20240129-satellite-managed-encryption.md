tags: []
---

# Satellite Managed Encryption

## Abstract

In order to provide a better experience for a large portion of our users, we want to support "satellite-managed encryption" for objects and paths. In short, this means that a project with satellite managed encryption:

* has all objects uploaded with encryption derived from a secure passphrase stored on the satllite and project salt
* has path encryption disabled

Supporting projects with these limitations means that it will be possible for users to create projects where they never need to enter a personally-managed encryption passphrase. Disabling path encryption also allows lexographical sorting by default, which is a feature that cannot be supported on projects with path encryption enabled.

## Background

We are talking about "satellite managed" encryption, but the code is organized such that generation of encryption details always happens on the client. In the Satellite GUI, encryption details are retrieved on the client side, and then inserted into an access via client-side WebAssembly. Similarly, the Uplink can insert encryption details during commands like `uplink setup` and `uplink access create`.

For a project flagged for satellite managed encryption, we need to:
* derive root encryption key for all accesses using a passphrase stored in a key-management store on the satellite and project salt
* create all accesses with the path cipher `storj.EncNull`

One example of a place that would need to be updated with these rules is [this part of the WebAssembly code](https://github.com/storj/storj/blob/a62929fd5757a40eda5d0b044ad2cefc14708410/satellite/console/consolewasm/access.go#L22-L29)

Because commands like `uplink setup` and `uplink access create` support the ability for a user to configure details like path encryption and encryption passphrase, it is important that for projects flagged as "satellite managed encryption", we never directly provide a user with a "raw API key". Rather, in the satellite UI, when the user creates credentials like access grants or S3 creds, these credentials should already be configured to use the satellite-managed encryption passphrase and no path encryption. This prevents the user from unintentionally creating a bad access in Uplink.

## Design

1. add a new `projects` DB column, like `projects.satellite_managed_encryption`, with all existing and new projects having this turned off
2. Key Management Store on satellite:
    * a secure way to associated project IDs with random encryption passphrases
    * should have heavy access restrictions and/or be encrypted
    * should be just as reliable as metainfo DB, since data loss or corruption from the KMS means data lost on the network
3. Satellite GUI changes:
    * support ability to create a project with "satellite managed encryption" turned on
    * any access credential created in the UI (for object browser, Uplink, or S3 tool) is configured to
        * use satellite KMS encryption passphrase + project salt for object encryption
        * have no path encryption enabled
    * ability to generate a "raw API key" in the Satellite UI is removed from satellite managed encryption projects
        * this is important for proper access usage for old versions of Uplink
4. Uplink changes:
    * new versions of Uplink can be updated to check for the new flag on the project, and automatically create accesses with the correct rules during commands like `uplink setup` and `uplink access create`

## Open issues

