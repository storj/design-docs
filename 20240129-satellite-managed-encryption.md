tags: ["satellite", "satellite/console", "encryption"]
---

# Satellite-Managed Encryption Passphrases

## Essentials

### Header

Date: 2024-01-29

Owner: mobyvb

Accountable:
- Console squad

Consulted:
- Product
- Delivery (TODO)
- Satellite (TODO)

Informed:
- Engineering

### Context

One common issue for new users is the added step of managing an encryption passphrase for a project. The user must store the encryption passphrase, and enter it when generating access credentials or using the object browser in the Satellite UI.

Having a user-managed encryption passphrase has privacy benefits for the user, but it adds an additional layer of complexity to user experience. We want to continue to support users who want to manage their own encryption credentials, while allowing other users who are not interested in managing their own encryption credentials to easily use the product. 

Additionally, by default, object paths are encrypted. This has privacy benefits, but prevents certain features that some users are interested in, like lexicographic ordering of object keys. Like with user-managed encryption passphrases, we want to continue supporting path encryption for users who are interested, while allowing other users who want lexicographic ordering features to create a project which has path encryption turned off.

### Goals

Users are provided the option to create a project that uses "satellite-managed encryption". One of these projects:
* uses a securely-stored, random passphrase stored on the satellite as the project's encryption passphrase
* restricts all files to be uploaded without path encryption

The main goals are to:
* provide users with an experience that does not require managing and entering an encryption passphrase in the Satellite UI
* provide users with an easy way to view and list files in lexographical order

### Approach / Design

#### Key Management Store

We need a secure way to store and look up project encryption passphrases on the Satellite. Someone with normal Satellite DB access should not be able to view this data, and ideally it is encrypted. At the same time, it must be very reliable, as losing data about project encryption passphrases means that data may have been lost. The primary features and concerns we should keep in mind:
* as simple as possible from an implementation perspective, e.g. key-value lookup for project ID -> encryption passphrase
* secure - should be encrypted, and highly restricted who/what has access to it
* reliable - should have the same standards for uptime/data loss as metainfo DB

##### Vault
The shortlisted KMS platform is [Hashicorp's Vault](https://www.vaultproject.io). Vault is a source-available identity-based secret and encryption management system that provides encryption services that are gated by authentication and authorization methods. Using Vault’s UI, CLI, or HTTP API, access to secrets and other sensitive data can be securely stored and managed, tightly controlled (restricted), and auditable ([source](https://developer.hashicorp.com/vault/docs/what-is-vault)).
Vault has a number of different feature like [encryption as a service](https://www.vaultproject.io/docs/secrets/transit), [key-value store](https://www.vaultproject.io/docs/secrets/kv), which more fits our use case, and more.

* Vault's encryption as a service is a high-level API for encrypting and decrypting data (passphrases in our case). It has no storage capabilities and will require us to store the encrypted passphrases, which we may not want to do because we want to as much as possible compartmentalise access to this data.
* Vault's key-value store is simply an encrypted storage for arbitrary key-value secrets, exactly as we need for our project ID -> encryption passphrase secrets.
It has a Golang library that can be used to trivially access our store instance from the Satellite backend.

##### Infisical
[Infisical](https://infisical.com) is an open source SecretOps platform that is used to securely store secrets and manage them across a development lifecycle. I can be used to manage secrets that access external infrastructure with a key feature that allows the user to have different values for secrets in different environments (development and production).
It can also be used to inject secrets during deployment. It is obvious that this platform is more designed for developer -> devops workflows, it can however be used to programmatically securely store project passphrases though it is not an intended uses case. It unfortunately does not have a Golang library.

For deployment, we have the option to host Vault/Infisical ourselves and manage it securely, or use a managed instances from Harshicorp/Infisical themselves. However, we choose to deploy it, we will need to ensure that it is highly available and secure so as not to leak or lose passphrases.
We must make sure to restrict access to deployment secrets.

To interact with the KMS, the satellite will have to be configured with;
* The address of the KMS
* Secret path (Where the secret is stored - [Vault](https://developer.hashicorp.com/vault/api-docs/secret/kv/kv-v1#path), It means the same for Infisical too)
* The authentication token used to initialise the vault client
* Or the Infisical access token and Infisical API key set in the `Authorization` and `X-API-KEY` header respectively in an http request.

It is important that these config values be mre protected than what is currently in storj/infra.

#### Satellite Project Updates

A new flag should be added to the `projects` table, like `projects.satellite_managed_encryption`. This should be false for all existing projects. While satellite-managed-encryption (this design) is in-progress, new projects should be created with this flag set to false.

Functionality should then be added to allow creating a project with a satellite managed encryption passphrase:
* the new column in the `projects` table is set to `true` for this project
* a new key-value pair is created for the new project in the Key Management Store. The passphrase should be cryptographically random
* endpoints on the satellite API are added or updated so that an authenticated user can request the encryption passphrase for a project during access creation

#### WebAssembly Updates

Wasm functionality must be updated to allow the Satellite UI to properly generate access credentials with path encryption disabled. [Code for reference](https://github.com/storj/storj/blob/a62929fd5757a40eda5d0b044ad2cefc14708410/satellite/console/consolewasm/access.go#L22-L29).

#### Satellite UI Updates

When creating a project, the user should be provided the option to create different types of projects:

* satellite-managed-encryption project
* user-managed-encryption project

The user-managed-encryption project will be the same as before, and user will always be prompted to enter encryption passphrase when necessary, e.g. when using the file browser or generating an access grant.

The satellite-managed-encryption project will never prompt the user for an encryption passphrase in the UI. Instead, when the user does an action that requires generating credentials, the UI will retrieve the project's encryption passphrase from the satellite KMS, and generate an access grant using this passphrase, and path encryption disabled.

The user should not be able to directly retrieve a "raw API key" from the Satellite UI. This prevents the user from unintentionally creating "invalid" credentials using a raw API key with Uplink. The only credentials that can be generated from the access page for a satellite-managed-encryption project should be credentials that already have embedded encryption rules.

The file browser in the UI should generate access credentials using the passphrase from the satellite KMS, and no path encryption.

#### Uplink Updates

New versions of Uplink can be updated to check for the new flag on the project, and automatically create accesses with the correct rules during commands like `uplink setup` and `uplink access create`.

This will allow new Uplink versions to properly handle "raw API keys" (access keys without information like satellite or encryption embedded) for "satellite-managed-encryption" projects.

These changes are technically unnecessary if the UI is updated to not provide raw API keys for satellite-managed-encryption projects. However, it may be useful to make these changes to Uplink so that future versions of Uplink can use raw API keys even for satellite-managed-encryption projects.

## Reminders

### Security / Privacy

The most important points to keep in mind:
* strict requirements for how the key-management store is built and accessed should be defined
* we should keep a path open for users who want to create projects using path encryption and user-managed encryption passphrases
* what happens if the project encryption passphrase is leaked? Is there a way to migrate data to use a new, secure passphrase, e.g. with server-side copy?

### Observability

We should ensure that we update analytics events during project creation to identify whether a project was created with satellite-managed or user-managed encryption.

### Test plan

The most important things to test:
* if a project is created with satellite-managed encryption, it should be very difficult or impossible for a user to unintentionally upload files with path encryption enabled, or with an encryption passphrase other than the project's satellite-managed passphrase.
* Satellite KMS for encryption passphrases is secure - should be encrypted, and access should be highly protected. Secure transfer of credentials to the client should only happen for project members.

### Rollout

This should be rolled out with a feature flag. The ability to create projects with satellite-managed-encryption should not be possible if the feature is turned off.

We should only turn the feature on when the UI and backend tasks in the design are all complete. It can be enabled before changes are made to Uplink, since the UI will not allow creating Uplink credentials with incorrect encryption settings.

### Rollback

In the event that we need to roll back the change, we should turn the feature flag off as soon as possible. Already-existing projects that have been created with satellite-managed-encryption should continue to work. The feature flag should only disable creation of such projects.

If the feature can be fixed in code, the fixes should be deployed before turning the feature flag back on.

If the feature cannot be fixed without user action, any user who has created a satellite-managed-encryption project should be contacted with instructions for how to access/migrate/handle any data they have uploaded.

## Open Questions
How to securely configure the satellite with secrets for the KMS? Currently, secrets in storj/infra are encrypted, but anyone with the access to the repo and certain permissions can access them. We need these to be more compartmentalised.

## Out of scope

