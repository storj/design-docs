tags: ["satellite", "satellite/console", "authentication", "sso"]
---

# General SSO MVP Design

## Essentials

### Header

Date: 2025-07-11

Owner: wilfred-asomanii

Accountable:
- Console team

### Context

Currently, We support configuring sign in via SSO provider for certain customers. When a user enters their email address,
we check if it matches an email pattern configured for an enterprise customer's SSO provider. If it does, we redirect the user to that provider for authentication.
We now want to expand this functionality to allow users to authenticate with general SSO providers, starting with Google.

General SSO will simplify the authentication process for users who prefer to use their existing accounts and may not have an
enterprise SSO provider configured for them. It will still maintain compatibility with existing enterprise SSO configurations
and email/password authentication.

### Goals

Users are provided the option to authenticate with "general SSO" providers. These authentication methods:
* use trusted third-party identity providers (Google for MVP)
* maintain compatibility with existing enterprise SSO configurations
* support hybrid authentication allowing continued use of email/password when desired

The main goals are to:
* provide users with an experience that does not require creating and managing passwords
* maintain enterprise SSO functionality with precedence over general providers
* provide a foundation for expanding to additional providers

### Approach / Design

#### Current Enterprise SSO

The existing Enterprise SSO implementation in `satellite/console/consoleauth/sso/` provides a foundation to build upon. Key parts include:

* **Provider configuration**: The `sso.OidcSetup` struct manages OIDC provider configurations with OAuth2 config and token verifiers
* **Provider selection by email**: The `(ssoService).GetProviderByEmail()` method uses regex patterns to match user emails to specific enterprise providers
* **External ID Storage**: The `users.external_id` column stores external identity provider user IDs in the form of `provider_name:external_user_id`
* **Authentication Flow**: Standard OAuth2 authorization code flow with callback handling via `VerifySso()`

The current system architecture supports multiple providers through configuration mappings, making it extensible for general providers.

#### Provider Type Classification

To support both enterprise and general SSO providers simultaneously, we need to classify providers by type.
Enterprise providers will always take precedence over general providers to maintain existing functionality.

The current `satellite/console/consoleauth/sso/service.go` implementation already supports multiple providers through
the `providerOidcSetup` map, but doesn't distinguish between provider types. The most straightforward approach is to
prefix the general provider names with a type identifier; e.g., `general-google` for Google SSO. This allows easy listing
of general providers and easy identification of which external ID belongs to which provider type.

#### Provider Resolution Flow

The existing implementation works as follows:
```
Email Input → Provider Resolution → Authentication Flow 
                   |                         | 
                   |                         ├── Password Authentication (if no match if found)
   Email Matching to configured providers    └── Configured Provider Authentication
```

With the introduction of General SSO, the flow will be extended to:
```
Email Input → Provider Resolution
                 |
                 ├── Enterprise Provider (if email matches) → Provider Authentication
                 └── General Provider (otherwise) → Authentication Flow Options
                                                       |
                                                       ├── Password Authentication
                                                       └── General SSO Provider Authentication
```

#### External ID

The existing `users.external_id` column in the database schema already supports distinguishing between different SSO providers
by storing the external user ID in a format like `provider_name:external_user_id`. This by default helps distinguish between
enterprise and general providers; general IDs will look like `general-google:google-user-id-456`, where `general-google` is
the general provider. Enterprise IDs may look like `storj-okta:user-guid-123`, same as they currently look.

#### Service Layer Architecture

The General SSO implementation will extend the existing SSO service in `satellite/console/consoleauth/sso/service.go`.
The current service already provides the foundation for provider management, token verification, and user authentication.

##### Current Service Structure Analysis

The existing `sso.Service` includes:
* `providerOidcSetup map[string]OidcSetup` - Provider configuration mapping
* `VerifySso()` method - Token verification and validation
* `Initialize()` method - Provider setup and configuration
* `GetProviderByEmail()` method - Email-based provider resolution
  * We will update this method to return an enterprise provider if applicable, or general providers otherwise.
```go
func (s *Service) GetProviderByEmail(email string) (enterprise string, general []string) {
	// 1. Check enterprise providers first 
	for provider, emailRegex := range s.config.EmailProviderMappings.Values {
		if emailRegex.MatchString(email) {
			return provider, nil
		}
	}
	// 2. Return general provider options 
	return "", ["general-google"]
}
```
Outside the `sso` package, we have to update certain parts of the Satellite that block access to SSO users, such as
change/reset password flows, and delete account flows. These should be updated to allow **general** SSO users to access these flows.
This does not include changing email.

##### Current Authentication Flow on the Satellite UI

The existing login flow in the satellite console follows this pattern:
1. User enters email address in the login/sign up form
2. System checks for enterprise SSO provider
3. If enterprise SSO is available, user is redirected to the provider's authentication page when they click "Continue"
   * By design, if the user logs in with an enterprise SSO provider, but the provided account is not linked to an existing
     satellite account, the satellite account with the same email address is linked to the SSO account. If there is no existing
     satellite account with that email, a new satellite account is created and linked to the SSO account.
4. Otherwise, user enters their password in the now visible password field.

For General SSO, we'll modify this flow to present SSO options alongside the password field when no enterprise provider is found.
The user will click any of the options, e.g., "Sign in with Google", which will redirect them to the OAuth2 flow for the
`general-google` provider. The flow remains largely the same, but with an additional step for provider selection and the following
differences:

* General SSO users will still be able to access the email/password authentication flow if they prefer.
* If the user logs in with a general SSO provider, and there is not an already linked satellite account with the same email,
  log in will fail.
* If the user signs up via a general SSO provider, and there already is a linked satellite account to that SSO account,
  the user will be logged in to that account.
* If the user signs up via a general SSO provider, and there is no (linked) satellite account to that SSO account,
  a new satellite account will be created and linked to the SSO account.
* If the user signs up via a general SSO provider, and there is an existing satellite account with the same email,
  sign up will fail.
* Account linking and unlinking will only be done from the account settings page for general SSO users.
  * Enterprise SSO users cannot unlink their SSO accounts or link to a General SSO provider.
* If a user logs in/signs up with an enterprise SSO provider, the matching satellite account will be linked to the SSO account
  regardless of whether it is already linked to a general SSO account or not.

##### UI Changes
The Satellite UI will need to be updated to support the new General SSO flow. The main changes include:
* **Login Page**: Update the login page to show a "Sign in with Google" button alongside the password field when no enterprise SSO provider is found.
* **Account Settings**: Allow general SSO users to manage their accounts fully, including setting a password, changing it, and deleting their account.
  * Have UI elements to link/unlink general SSO accounts, IF there are General SSO providers available (just Google for now).

#### API Endpoints
**GET /api/v0/auth/sso/url?email=user@example.com**

This endpoint will be updated to return an enterprise SSO URL if the email matches an enterprise provider,
or a list of general SSO providers if no enterprise match is found.
```go
type SSOProvidersResponse struct {
    Enterprise string   `json:"enterprise,omitempty"`
    General    []string `json:"general"`
}
```

##### Updated SSO Flow (General SSO)

###### Linking from Account Settings

1. User signs up with email and password.
2. User logs in to the new account with email and password.
3. User goes to account settings and clicks "Link Google Account".
4. If the user has MFA enabled, they will be prompted for it.
5. UI makes request to `GET /api/v0/auth/sso/url?email=user@example.com` for the Google SSO URL.
6. User is redirected to the Google OAuth2 flow.
7. On successful authentication, the user is redirected back to the Satellite with an authenticated session.
8. User can now subsequently log in with the Google SSO account.

###### Sign Up with General SSO

1. User enters email address in the sign up form.
2. UI makes request to `GET /api/v0/auth/sso/url?email=user@example.com` for SSO URLs.
3. If no enterprise SSO provider matches the email, the UI shows a "Sign up with Google" button if that is available.
4. User clicks "Sign up with Google" and gets redirected to the Google OAuth2 flow.
5. On successful authentication, the user is redirected back to the Satellite with an authenticated session.
6. User can now go to account settings to set a password.


#### Configuration
The current SSO configuration is as such;
```go
type Config struct {
	...
	OidcProviderInfos     OidcProviderInfos     `help:"semicolon-separated provider:client-id,client-secret,provider-url." default:""`
	EmailProviderMappings EmailProviderMappings `help:"semicolon-separated provider:email-regex as provided in oidc-provider-infos." default:""`
	...
}
```
All we have to do is to add general SSO provider configurations to `OidcProviderInfos`. E.g.;
```yaml
STORJ_SSO_OIDC_PROVIDER_INFOS: general-google:client-id,client-secret,https://accounts.google.com;storj-okta:client-id,client-secret,https://okta-domain.com
```
We do not need to change the `EmailProviderMappings` configuration, as general providers will not use email matching.

**NB**: having no provider with the "general-" prefix effectively disables general SSO functionality.

#### What to Test
- **Google OAuth2 flow**: we have to make sure the Google SSO flow works correctly;
  - link a satellite account to a Google SSO account
  - log in to the UI with the Google SSO account
  - unlink the Google SSO account from the satellite account
  - log in with the Google SSO account again should fail
  - sign up with that same Google SSO account should fail
  - sign up with a different Google SSO account should create a new satellite account
- **Enterprise SSO flow**: test that all existing enterprise SSO flows continue to work as expected.
- **Enterprise SSO precedence**:
  - a user with an email matching an enterprise SSO provider should always be redirected to that provider, even if general SSO is available.
  - a user already linked to an enterprise SSO provider should not be able to log in with General SSO or password.
  - a user already linked to an enterprise SSO provider should not be able to unlink from the provider.
  - a user with an email matching an enterprise SSO provider should not be able to log in with general SSO or password.
- Password reset/change functionality
- Account deletion functionality

## Implementation Tickets

1. **[SATELLITE] Extend SSO Service for General Providers**
  - Modify the code in `satellite/console/consoleauth/sso/service.go` to support general SSO providers
  - General providers are identified by the "general-" prefix.
  - Update the /api/v0/auth/sso/url endpoint to return general SSO options when no enterprise provider matches the email.
  - Update restrictions to allow general SSO users to access functionality current SSO are not allowed to access,
    such as change/reset password and delete account flows, except changing email.

2. **[SATELLITE UI] Update Login/Sign up Flow**
  - For each general SSO provider, received from the /sso/url endpoint,
    add a button that redirects to that provider's authentication flow.

3. **[SATELLITE UI] Allow general SSO users to manage their account fully**
  - Allow users to link/unlink general SSO accounts from account settings.
  - If a user is linked to a general SSO provider,
    - Show a set password option in account settings if they have no password
    - Show the existing change password option otherwise.
    - Allow them to delete their account.

4. **Set up Google SSO**
  - Set up an external Google OAuth2 application in GCP
  - Add the client ID and secret to the `STORJ_SSO_OIDC_PROVIDER_INFOS` configuration

## Open Questions
* Should we allow general SSO users to change their email?
* Should we store state tokens in the db instead of cookies?
* Should we require extra validation for linking a general SSO account to an existing user?
  * What happens if a user's SSO provider account is compromised? Account linking should be secure and reversible

## Out of Scope
This MVP focuses on Google SSO as the only general provider. Setting up other providers is trivial since the architecture
is already in place. We only need to add the provider configuration to the `OidcProviderInfos` and update the UI accordingly
to show the new provider options.