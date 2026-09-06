# Reading more than one tenant

A dashboard can span tenants — but only from the right render element, and only with one token
per tenant.

## Tokens are per-tenant. There is no estate-wide token.

Exchange the viewer's session token for a tenant-scoped one, exactly as the Enterprise Data
Explorer does:

```
POST <api>/authentication
Authorization: Bearer <the viewer's token>

mutation { data: initiateAuthenticationFlow(payload: {
  flowType: "TokenRefresh",
  flowData: { token: <viewer token>, tenantId: <target tenant> }
}) { token challenge { type } } }
```

A tenant the viewer cannot reach returns no token. That is an access answer, not an error —
skip that tenant and carry on.

## This requires a same-origin render element

The page reads `parent.localStorage.token` and `parent.localStorage.tenantId`, which only works
**same-origin**. A `DisplayText` `srcdoc` page is same-origin; an `EmbeddedWebpage` pointed at a
`data:` URI is not.

> If your page reports "no session token reachable", the render element is wrong — not the code.

## What `metadata` will and will not give you

`$metadata` exposes `{ tenant, enterprise, userTenants, userRoles, currentRole, user }`.

- **`userTenants` is only the tenants this user can reach** — not the estate.
- The full list is `FuuzEnterpriseEnvironmentTenant` in the administration enterprise, which
  ran to **662 rows** across all enterprise/environment pairs when last measured.

## Tenants do not share a schema

This is the constraint that shapes the whole design. A model present in one tenant may be absent
in ten others — `workunit` existed in **1 of 11** tenants in the estate that was measured. So:

- Query defensively. A missing model is normal, not a failure.
- Build the cross-tenant view from what is **common** (applications, flows, screens, models),
  and drill into plant detail **per tenant**, where the schema supports it.

## Where the tokens belong

A page cannot authenticate itself — it can only exchange a token a signed-in viewer already has.
That makes page-side fan-out fine for a **per-user** estate view, where every tenant shown is one
the viewer could open anyway.

For an estate view that must show tenants **regardless of who is looking**, the per-tenant
credentials belong server-side as `Connection` records, with a flow doing the fan-out. Do not
try to put estate-wide credentials in the page.
