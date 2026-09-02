---
name: venafi-request-tls-certificate
description: Request a TLS certificate from Venafi / CyberArk Certificate Manager - SaaS against an
  application and issuing template, then poll for issuance and download the chain.
api: openapi/venafi-certificate-manager-saas-openapi.yml
generated: '2026-09-02'
method: generated
source: openapi/venafi-certificate-manager-saas-openapi.yml,
  https://developer.venafi.com/tlsprotectcloud/reference/tls-protect-overview
operations:
  - applications_getAll
  - certificateissuingtemplate_getAll
  - certificaterequests_validation
  - certificaterequests_create
  - certificaterequests_getById
  - certificates_getContentsById
---

# Request a TLS certificate

Base URL is regional. `https://api.venafi.cloud` is the US default; the others are
`api.eu`, `api.uk`, `api.au`, `api.sg` and `api.ca` under `venafi.cloud`. Use the one that matches
the tenant's data residency — a request to the wrong region will not find the tenant.

Every call carries `Content-Type: application/json` and either `tppl-api-key: <user API key>` or a
service-account bearer token. HTTPS is required.

## 1. Find the application and issuing template

`GET /outagedetection/v1/applications` (`applications_getAll`) lists the applications the caller can
see. Certificates hang off an application, so you need its id before anything else.

`GET /v1/certificateissuingtemplates` (`certificateissuingtemplate_getAll`) lists the templates.
The template is the policy: it constrains subject fields, SANs, key algorithm, validity and which CA
issues. Pick the template alias the application is configured for.

## 2. Validate before you commit

`POST /outagedetection/v1/certificaterequests/validation` (`certificaterequests_validation`) takes
the same body as the create call and evaluates it without submitting. Do this first. It is the only
rehearsal this API gives you, and the alternative is a rejected request that has already consumed a
CA order.

Read the response as `{"errors":[{"code":<int>,"message":"...","args":[...]}]}`. The numeric `code`
is stable — branch on it, not on the message. `errors/venafi-problem-types.yml` in this repository
lists 90 documented codes.

## 3. Submit

`POST /outagedetection/v1/certificaterequests` (`certificaterequests_create`). Supply either a CSR
you generated, or the subject/SAN fields and let the service generate the key pair.

**There is no idempotency key.** Nothing in this contract makes a retried POST replay-safe. If the
call times out, do not blindly retry — call `certificaterequests_getById` or search by common name
first, or you will issue a duplicate certificate and consume a second CA order.

## 4. Poll for issuance

`GET /outagedetection/v1/certificaterequests/{id}` (`certificaterequests_getById`) until the request
reaches an issued state. Issuance is asynchronous; how long it takes depends on the CA and on
whether an approval rule is in play. If an approval rule applies, the request waits for a decision
from `certificaterequests_approve`.

## 5. Download

`GET /outagedetection/v1/certificates/{id}/contents` (`certificates_getContentsById`) returns the
certificate. Query parameters control `format`, `chainOrder` and whether the chain is included; the
response is `application/octet-stream` or `text/plain` depending on format, not JSON.

## Undoing this

Issuance cannot be un-issued. The two paths back are:

- **Retire** the certificate — `POST /outagedetection/v1/certificates/retirement` — which is
  reversible with `POST /outagedetection/v1/certificates/recovery`. `POST
  /outagedetection/v1/certificates/deletion` is permanent and only accepts already-retired
  certificates.
- **Revoke** it, which invalidates it with the CA. That is mitigation, not reversal.

Neither the contract nor the docs state a window between retirement and permanent deletion, so do
not promise a caller one.
