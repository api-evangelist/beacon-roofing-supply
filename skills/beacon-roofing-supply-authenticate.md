---
name: beacon-authenticate
description: >-
  Obtain, use, refresh and troubleshoot credentials for the Beacon/QXO Beacon Rest Services — the
  OAuth bearer flow for V2/V3, the V1 cookie session, the apiSiteId multi-site selector, and the
  permission model that decides what your token may actually call.
api: Beacon Rest Services (Beacon PRO+)
provider: beacon-roofing-supply
base_url: https://beaconproplus.com/rest/model/REST/oauth
spec: openapi/beacon-roofing-supply-oauth2-openapi.yml
generated: '2026-09-04'
method: generated
source: >-
  Grounded in openapi/beacon-roofing-supply-oauth2-openapi.yml, the securitySchemes and
  info.description of the V1/V2/V3 documents, and live unauthenticated probes of
  https://beaconproplus.com/v1|v2|v3/rest/com/becn/* on 2026-09-04
operations:
  - V1Controller_login
  - V1Controller_logout
  - V1Controller_getLoginDeclaration
  - V1Controller_accounts
  - V1Controller_switchAccount
  - V2Controller_getCurrentUserInfo
  - V2Controller_getCurrentUserPermission
  - V2Controller_permissionTemplateList
  - V2Controller_getPermissionTemplateDetail
---

# Authenticate against Beacon (QXO)

## Getting credentials at all

You cannot self-serve. The published contract documents only the **refresh_token** grant — there is
no authorization_code, client_credentials or password flow anywhere on the public surface, which
means the first `refresh_token` must be issued to you.

1. Be an existing QXO customer. If not: https://www.qxo.com/open-an-account
2. Request API access: https://go.qxo.com/qxoapi
3. Accept the API license terms: https://www.qxo.com/integrations/api-license-terms

The six products QXO sells against this API — Order, Pricing, Account, Product, Delivery Tracking,
Invoice — are described at https://www.qxo.com/customapi. Nothing is priced publicly.

## Two authentication models, by version

| Surface | Mechanism | Scheme |
|---|---|---|
| V2, V3, Public, Internal | OAuth 2.0 bearer token | `Authorization: Bearer <access_token>` |
| V1 (and v1_ng, v2_ng) | Cookie session | `JSESSIONID`, `DYN_USER_ID`, `DYN_USER_CONFIRM`, `siteId`, `rememberPassword` |

The cookie model is a browser session carried into an API contract. `V1Controller_login` establishes
it; the session lasts **1 hour**, extended to **7 days** with the `RememberMe` flag. `siteId` is
always `homeSite` on login. `V1Controller_logout` ends it. `V1Controller_getLoginDeclaration` returns
the login declaration.

Prefer V2/V3 and the bearer token for anything programmatic.

## Refreshing the token

`POST https://beaconproplus.com/rest/model/REST/oauth/token`, body `application/json`:

```json
{
  "grant_type": "refresh_token",
  "refresh_token": "<your refresh token>",
  "client_id": "<your client id>",
  "scopes": "all"
}
```

Response: `access_token`, `token_type`, `expires_in`, `refresh_expires_in`, `refresh_token`, `scope`,
`success`, `messages`.

**`scopes` is not a permission scope.** It is a refresh-behaviour selector, and Beacon documents
exactly three behaviours:

| Value | Effect |
|---|---|
| *(empty)* | Refresh the `access_token` only |
| `all` | Refresh both `access_token` and `refresh_token` |
| `refresh_token` | Also refresh both |

Multiple values are whitespace-separated. Note there is **no `client_secret`** in the published
request. Treat the `refresh_token` itself as the secret and store it accordingly.

Token endpoint failures come back keyed by field: `grant_type` → "Bad Grant Type"; `refresh_token` →
"Empty Refresh Token" / "Empty Request" / "Empty Refresh Token or Client ID" / "Invalid Token" /
"Expired Token"; `client_id` → "Empty Client ID"; `token_validation` → "Get token failed" /
"Validate token failed".

Note the OAuth document's `servers[]` enum defaults to **`beacon-uat.becn.com`**, not production.
If you copy the base URL out of the Swagger UI you will be pointed at UAT — which is safe, but is
also unreachable from the public internet.

## 401 vs 419 — do not collapse these

- **401** — the token is *invalid*, or on V1 the user is not logged in. Re-authenticate.
- **419** — the bearer token is *expired*. 154 operations declare it. **419 is not a registered IANA
  HTTP status code**, so a generic HTTP client will treat it as an unknown 4xx. Handle it explicitly:
  refresh the token and retry the request.

Verified live 2026-09-04, an unauthenticated call returns a structured JSON body rather than a bare
status — e.g. `GET /v3/rest/com/becn/branchData` →
`{"success":false,"messages":{"code":401,"value":"Please provide token"},"code":401,"traceId":"…"}`,
while V1 returns `{"success":false,"messages":[{"type":"error","value":"Please provide token","code":2004}],"traceId":"…"}`.
Note the shapes differ between versions: V3 uses an object for `messages`, V1 an array. Parse
defensively.

## Choose the account you are acting for

A profile can act for several accounts:

- `V1Controller_accounts` — the accounts available.
- `V1Controller_switchAccount` — switch the working account.
- `V2Controller_getCurrentUserInfo` — who you currently are.

Profile-level failures: **2001** profile empty, **2003** profile has no available accounts, **2004**
profile unauthorized, **2006** permission denied. Account-level: **3001** account not yours, **5006**
"Need change account", **5007** "Account no longer available for current user", **5009** "Account
token invalid".

## Discover what your token may do

There is **no scope string and no capability-discovery endpoint**. Authorization is a server-side
permission template. The only way to know your surface without hitting a 403:

- `V2Controller_getCurrentUserPermission` — the current profile's permissions.
- `V2Controller_permissionTemplateList` / `V2Controller_getPermissionTemplateDetail` — the templates.

102 operations declare a 403 ("Forbidden, user do not has permission to access this API"); eleven of
those narrow it to master-admin or admin users only. Call `getCurrentUserPermission` at start-up and
gate your own tool surface on the result rather than discovering it by failure.

## apiSiteId — the multi-site selector

Most operations accept `apiSiteId` (example value `"DEN"`). Resolution order, highest first:

1. `apiSiteId` in the query string.
2. `apiSiteId` as a **root** element of the JSON request body. A nested `apiSiteId` is ignored.
3. The site pre-bound to your token's `client_id` in Beacon's database.

Because of (3), omitting `apiSiteId` does not mean "unset" — it means "the site my credentials belong
to". If you serve several Beacon sites from one integration, set it explicitly every time.

## Transport

TLS 1.3 with `strict-transport-security: max-age=31536000; includeSubDomains` on the API host,
confirmed 2026-09-04. No `/.well-known/openid-configuration` or
`/.well-known/oauth-authorization-server` is served on any Beacon or QXO host — the token endpoint is
at a vendor path and is not discoverable. There is no OIDC.
