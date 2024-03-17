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

The KMS will be made up of 3 components:
* The master key and store - The master key will be used to encrypt and decrypt passphrases. The store will be used to securely store the master key. The store will be [Google's Secret Manager](https://cloud.google.com/security/products/secret-manager), which is a secure store for secrets such as API keys and other sensitive data.
This secrets manager was chosen because it is just as secure as the alternatives, if not more and yet cheaper and easier to manage. It also is more convenient to use as we already use GCP.
* The KMS service - This is part of the Satellite that will be responsible for fetching the master key at startup, encrypting and decrypting passphrases, and storing and retrieving same from the passphrase store. The master key will be kept in memory throughout the lifetime of the Satellite.
The satellite will be configured with credentials needed to access the Secret Manager (the master key store).
* The Passphrase store - This where the encrypted passphrases will be stored. It will be a new column, `passphrase_enc` on the `projects` table. As added benefits to storing this on `projects`, we will be able to back up encrypted passphrases and also rotate master keys.

#### Satellite Project Updates

A new column should be added to the `projects` table to be the passphrase store, like `projects.passphrase_enc`. This should be the encrypted passphrase for satellite-managed-encryption projects only and `null` for others.
Projects that are created before this feature is deployed and projects created after but opt out should have `null` in this column.

Functionality should then be added to allow creating a project with a satellite managed encryption passphrase:
* the KMS service generates a cryptographically random passphrase and encrypts it using the master key
* the passphrase store is updated with the encrypted passphrase for this project
* endpoints on the satellite API are added or updated so that an authenticated user can request the encryption passphrase for a project during access creation
  * endpoints handlers should use the KMS service to retrieve the encrypted passphrase from the store and decrypt it using the master key

#### Satellite UI Updates

When creating a project, the user should be provided the option to create different types of projects:

* satellite-managed-encryption project
* user-managed-encryption project

The user-managed-encryption project will be the same as before, and user will always be prompted to enter encryption passphrase when necessary, e.g. when using the file browser or generating an access grant.

The satellite-managed-encryption project will never prompt the user for an encryption passphrase in the UI. Instead, when the user does an action that requires generating credentials, the UI will retrieve the project's encryption passphrase from the satellite KMS, and generate an access grant using this passphrase, and path encryption disabled.

The user should not be able to directly retrieve a "raw API key" from the Satellite UI. This prevents the user from unintentionally creating "invalid" credentials using a raw API key with Uplink. The only credentials that can be generated from the access page for a satellite-managed-encryption project should be credentials that already have embedded encryption rules.

The file browser in the UI should generate access credentials using the passphrase from the satellite KMS, and no path encryption.

#### External Updates

###### WebAssembly and Bindings

* Wasm functionality must be updated to allow the Satellite UI to properly generate access credentials with path encryption disabled. [Code for reference](https://github.com/storj/storj/blob/a62929fd5757a40eda5d0b044ad2cefc14708410/satellite/console/consolewasm/access.go#L22-L29).
* Our bindings for other languages should also be reviewed and possibly updated to work properly as well.
  * [Python access library](https://github.com/storj/access-python) 

###### Uplink

New versions of Uplink (uplink-cli, libuplink and the uplink-c) can be updated to check for the new flag on the project, and automatically create accesses with the correct rules during commands like `uplink setup` and `uplink access create`.

This will allow new Uplink versions to properly handle "raw API keys" (access keys without information like satellite or encryption embedded) for "satellite-managed-encryption" projects.

These changes are technically unnecessary if the UI is updated to not provide raw API keys for satellite-managed-encryption projects. However, it may be useful to make these changes to Uplink so that future versions of Uplink can use raw API keys even for satellite-managed-encryption projects.

## Reminders

### Security / Privacy

The most important points to keep in mind:
* strict requirements for how the key-management store is built and accessed should be defined
* we should keep a path open for users who want to create projects using path encryption and user-managed encryption passphrases
* what happens if the project encryption passphrase is leaked? Is there a way to migrate data to use a new, secure passphrase, e.g. with server-side copy?
* we should have a clear UI cues for users to understand the implications of creating a project with satellite-managed encryption

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

### Alternative KMS approaches
One way we could go about achieving this is to fully delegate the storage of encrypted passphrases to any of these third parties;

##### Vault
[Hashicorp's Vault](https://www.vaultproject.io) is a source-available identity-based secret and encryption management system that provides encryption services that are gated by authentication and authorization methods. Using Vault’s UI, CLI, or HTTP API, access to secrets and other sensitive data can be securely stored and managed, tightly controlled (restricted), and auditable ([source](https://developer.hashicorp.com/vault/docs/what-is-vault)).
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

Another approach we can take for this is encrypting and storing passphrases in satellite DB to reduce reliance on third party services.
Different options have been considered and common among them is encryption, where we will use a master key to encrypt passphrases. Another common aspect is storage; the encrypted passphrases will be stored in a new column on the projects table.

##### Option 1: No third party
For this option, it has been decided that the satellite config is secure enough to keep the master key in.
A new config will be added to the satellite and will be used to encrypt and store passphrases as described above. In this case, we will have no reliance on any third party service. Adversely, it might not be as secure to keep the master key in config.

##### Option 2: Use Vault
This option is exactly the same as what is described in the Key Management Store section above. The only difference is that we will be using Vault to store the master key.
A downside to this as compared to Google's Secret Manager is that it is more expensive and more cumbersome to manage. Also, we already use GCP and it is more convenient to use one more service.

## Open Questions
* How to securely configure the satellite with the encryption master key or credentials to access Secrets Manager?
* How to securely back up and/or rotate the master key?

## Out of scope

