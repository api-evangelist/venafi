---
name: venafi-authenticate-websdk-oauth
description: Obtain, refresh and revoke an OAuth 2.0 access token against a self-hosted Venafi /
  CyberArk Trust Protection Foundation instance, and work out which scope a call needs.
api: openapi/venafi-trust-protection-foundation-websdk-openapi.yml
generated: '2026-09-02'
method: generated
source: openapi/venafi-trust-protection-foundation-websdk-openapi.yml,
  https://docs.venafi.com/Docs/currentSDK/TopNav/Content/SDK/AuthSDK/r-SDKa-OAuthScopePrivilegeMapping.php
operations:
  - Venafi_Web_SDK_Authentication_Authorize_AuthorizeOAuth
  - Venafi_Web_SDK_Authentication_Authorize_Token
  - Venafi_Web_SDK_Authentication_Revoke_RevokeGrant
  - Venafi_Core_WebSDK_RecycleBinRest_Restore
---

# Authenticate against a self-hosted Trust Protection Foundation

This API has **no vendor host**. It runs inside the customer's network and the contract says so: its
server is `https://{dnsname}/`, a template. Ask the operator for the hostname; never guess one, and
ignore the literal `https://REPLACEdnsnameME/` entry that escaped into the published spec.

## Get a token

`POST /vedauth/authorize/oauth` (`Venafi_Web_SDK_Authentication_Authorize_AuthorizeOAuth`) is the
authorization-code flow. Four alternatives exist for non-interactive callers:

| Endpoint | Use |
|---|---|
| `POST /vedauth/authorize/device` | device grant, for input-constrained clients |
| `POST /vedauth/authorize/jwt` | JWT bearer — federate from an external OIDC issuer |
| `POST /vedauth/authorize/certificate` | client X.509 certificate |
| `POST /vedauth/authorize/integrated` | integrated Windows authentication |

Refresh with `POST /vedauth/authorize/token`
(`Venafi_Web_SDK_Authentication_Authorize_Token`). The provider's guidance is to reuse one token
until it expires, track the expiry, and refresh rather than re-authorize.

Send it as `Authorization: Bearer <token>` on every `/vedsdk/` call.

## Work out the scope first

**Scopes are documented in prose, not in the contract's security definitions.** The only
`securityScheme` is `AccessToken: {type: http, scheme: bearer}` — a tool cannot compute the scope
you need from the spec. Instead, each operation's *description* ends with
`_Required scope: <scope>_`.

Scopes take the form `<scope>:<privilege>`, and a client declares the union it needs at
authorization time, e.g. `scope: certificate:discover,delete,manage,revoke`. A bare scope name
grants read. Whatever you ask for, you also implicitly receive the `any` scope, which covers
read-only system, config-lookup, metadata and log endpoints.

`scopes/venafi-scopes.yml` in this repository normalises all 33 published values with the number of
operations that require each.

Two of them are malformed in the contract itself: 11 operations publish
`_Required scope: :manage_` and 4 publish `_Required scope: :approve_`, with an empty prefix. The
provider's scope map resolves these to the `any` scope carrying the Manage or Approve privilege
(`POST Log`, `POST Metadata/Set`, `POST Workflow/Ticket/UpdateStatus`).

## Give the token back

`GET /vedauth/revoke/token` (`Venafi_Web_SDK_Authentication_Revoke_RevokeGrant`) revokes the grant.
Do this on exit. A long-lived unrevoked grant on a certificate-authority platform is exactly the
kind of standing credential this product exists to eliminate.

## If you delete the wrong thing

`POST /vedsdk/recyclebin/restore` (`Venafi_Core_WebSDK_RecycleBinRest_Restore`) restores a deleted
item. It needs the `admin:recyclebin` scope, which means the token you used to delete probably
cannot restore. Ask for both up front if the workflow includes deletes. No retention window is
published for the recycle bin.
