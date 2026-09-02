---
name: venafi-retire-and-recover-certificates
description: Retire certificates from the Venafi / CyberArk Certificate Manager - SaaS inventory,
  recover them if the retirement was wrong, and understand which step is permanent.
api: openapi/venafi-certificate-manager-saas-openapi.yml
generated: '2026-09-02'
method: generated
source: openapi/venafi-certificate-manager-saas-openapi.yml
operations:
  - certificateretirement_retireCertificates
  - certificateretirement_recoverCertificates
  - certificateretirement_deleteCertificates
  - certificates_search_getByExpression
---

# Retire, recover, delete

Three operations, and only the middle one is a way back.

## Retire (reversible)

`POST /outagedetection/v1/certificates/retirement`
(`certificateretirement_retireCertificates`) takes a `certificateIds` array and retires each one.
Retirement takes a certificate out of active inventory and monitoring. It does **not** revoke the
certificate — a retired certificate that is still installed somewhere keeps working until it
expires. If the intent is "make this certificate stop being trusted", retirement is the wrong
operation; revocation is.

## Recover (the reversal)

`POST /outagedetection/v1/certificates/recovery`
(`certificateretirement_recoverCertificates`) recovers the certificates named by `certificateIds`,
**including any previous versions of those certificates** — that is the provider's own wording, and
it matters: recovery restores the version history, not just the current certificate.

The provider does not publish a window for this. Nothing in the contract or the documentation says
how long a retired certificate stays recoverable. Do not tell a caller they have N days.

## Delete (permanent)

`POST /outagedetection/v1/certificates/deletion`
(`certificateretirement_deleteCertificates`) "permanently deletes the retired certificates
specified by `certificateIds` from the inventory." It only accepts certificates that are already
retired, which is the one piece of safety in the design: you cannot delete something in a single
call, you have to retire it first.

Treat this as an operation that requires a human decision. There is no undo, no recycle bin on the
SaaS side (the recycle bin, `POST /vedsdk/recyclebin/restore`, belongs to the self-hosted WebSDK),
and no idempotency key — so a retried delete that partially succeeded leaves you with no way to
tell what was lost.

## Batch shape

All three take an array. Build the id list from
`POST /outagedetection/v1/certificatesearch` and **print the count and a sample before
executing**. A malformed expression that matches the whole tenant will retire the whole tenant.
