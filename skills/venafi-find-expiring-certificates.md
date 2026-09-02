---
name: venafi-find-expiring-certificates
description: Search the Venafi / CyberArk Certificate Manager - SaaS inventory for certificates
  expiring inside a window, using the expression-tree search API with paging.
api: openapi/venafi-certificate-manager-saas-openapi.yml
generated: '2026-09-02'
method: generated
source: openapi/venafi-certificate-manager-saas-openapi.yml,
  https://docs.venafi.cloud/api/searching-for-expiring-certificates/
operations:
  - certificates_search_getByExpression
  - certificates_getById
  - certificateinstances_search_getByExpression
---

# Find expiring certificates

This is the single most common reason to call this API, and it is a POST, not a GET.

## The search shape

`POST /outagedetection/v1/certificatesearch` (`certificates_search_getByExpression`). The body has
three parts:

```json
{
  "expression": { "operator": "AND", "operands": [ ... ] },
  "ordering":   { "orders": [ { "field": "validityEnd", "direction": "ASC" } ] },
  "paging":     { "pageNumber": 0, "pageSize": 100 }
}
```

`expression` is a recursive tree. A **compound** node carries an `operator` of `AND`/`OR` and nests
children in `operands`. A **leaf** node applies a comparison `operator` to a `field` with `value`
(or `values` for set operators). To find certificates expiring in the next 30 days, build a leaf on
the validity-end field with a `LTE` comparison against an ISO-8601 instant, and AND it with whatever
scoping you need (application, tag, certificate status).

## Paging

There is no cursor and no `Link` header. You page by incrementing `paging.pageNumber` with a fixed
`paging.pageSize` until a page comes back short. Because you are paging a live inventory ordered by
expiry, a certificate can move between pages while you iterate — order by a stable field and
de-duplicate by certificate id if exactness matters.

## Rate limiting

None is published. There is no `429` response declared anywhere in this contract, no
`RateLimit-*` header and no `Retry-After`. That is not permission to hammer it: treat any 5xx as a
signal to back off with jitter, because you have no runtime signal telling you when you are close
to a ceiling.

## Installations, not just certificates

A certificate in inventory is not the same thing as a certificate installed somewhere.
`POST /outagedetection/v1/certificateinstancesearch`
(`certificateinstances_search_getByExpression`) searches the discovered *installations* — host,
port, and which certificate is serving there. If the question is "what will break when this
expires", this is the search you want, not the inventory one.
