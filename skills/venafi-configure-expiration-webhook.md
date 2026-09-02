---
name: venafi-configure-expiration-webhook
description: Configure certificate-expiration notifications from Venafi / CyberArk Certificate
  Manager - SaaS to a Slack or generic webhook, including the global monitoring thresholds.
api: openapi/venafi-certificate-manager-saas-openapi.yml
generated: '2026-09-02'
method: generated
source: openapi/venafi-certificate-manager-saas-openapi.yml,
  https://developer.venafi.com/tlsprotectcloud/docs/control-plane-receive-webhook-notifications
operations:
  - inventorymonitoringconfiguration_getByType
  - inventorymonitoringconfiguration_update
  - connectors_create
  - connectors_getAll
  - connectors_delete
---

# Send expiring-certificate notifications to a webhook

Two objects have to agree: the **monitoring configuration** decides *when* a notification fires, and
the **connector** decides *where* it goes. Configuring one without the other produces silence.

## 1. Read the current monitoring configuration

`GET /outagedetection/v1/inventorymonitoringconfig/CERTIFICATE_EXPIRATION`
(`inventorymonitoringconfiguration_getByType`). Read before you write — this configuration is
tenant-global and it drives email notifications too, so an update here changes behaviour for people
who did not ask for it.

## 2. Update it

`PUT /outagedetection/v1/inventorymonitoringconfig/CERTIFICATE_EXPIRATION`
(`inventorymonitoringconfiguration_update`) with an `inventoryMonitoringConfiguration` object:

- `enabled` — boolean.
- `thresholds` — days before expiry at which to notify. **The provider limits this to 3 values.**
- `applicationIds` — which applications this applies to. Empty means all.
- `includeUnassignedCertificates` — whether certificates with no application are covered.

Certificates in applications that are not monitored will not notify, even if the application is
named on the connector. The provider explicitly recommends monitoring all applications.

## 3. Create the connector

`POST /v1/connectors` (`connectors_create`) with:

- `connectorKind: WEBHOOK`
- `filterType: EXPIRATION`
- `type` — `slack` or `generic`
- `url` — the incoming webhook URL
- `applicationIds` — leave empty to inherit the global monitoring settings. If the monitoring
  configuration names specific application ids, the ids here **must overlap** or nothing fires.

**The tenant is limited to 10 webhook connections.** Check `GET /v1/connectors`
(`connectors_getAll`) before creating one, and clean up with `DELETE /v1/connectors/{id}`.

## What you do not get

There is no signing secret, no signature header, no delivery log and no documented retry policy.
A receiver cannot verify that an inbound POST came from Venafi using anything the provider
publishes. Terminate the webhook on an endpoint that is hard to guess and treat the payload as
untrusted.
