---
name: docontrol-authenticate
description: Exchange a DoControl API key for a short-lived bearer access token and call the GraphQL gateway with it.
api: docontrol:docontrol-authentication-api
operations:
  - refreshToken
  - graphql
generated: '2026-09-06'
method: generated
source: https://docs.docontrol.io/docontrol-user-guide/system-management/api.md
---

# Authenticate against the DoControl API

Every DoControl API call needs a bearer access token that lives for five minutes. You get one by
exchanging the long-lived API key (DoControl calls it a *refresh token*) that an admin minted in
the console.

## Before you start

- You need an API key with at least **admin** privilege. A `viewer` key cannot call the API.
- The key is shown **once**, at creation, in Settings > Admin > API keys. It cannot be read back.
- A tenant may hold at most **10** keys.

## Step 1 — exchange the key (`refreshToken`)

```
POST https://auth.prod.docontrol.io/refresh
Content-Type: application/json

{"refreshToken": "<YOUR_DOCONTROL_API_KEY>"}
```

The response carries three fields:

```json
{"token": "<access token>", "expiresIn": 300, "refreshToken": "<refresh token>"}
```

Take **`token`** — that is the five-minute access token. Do not send `refreshToken` as a bearer
credential; it is the long-lived key you keep in your secret store.

> The OpenAPI in `openapi/` models this response as `{accessToken, expiresIn}`. DoControl's own
> published example returns `token`. Read `token`, and fall back to `accessToken` only if `token`
> is absent. See `overlays/docontrol-authentication-api-overlay.yaml`.

## Step 2 — call the gateway (`graphql`)

```
POST https://apollo-gateway-v4-api.prod.docontrol.io/graphql
Authorization: Bearer <token>
Content-Type: application/json

{"query": "{ __typename }"}
```

## Handling the five-minute expiry

`expiresIn` is 300 seconds. Any run longer than that must re-exchange mid-flight. The pattern
DoControl uses in its own MCP server is to refresh when fewer than two minutes remain rather than
waiting for a failure.

## Error handling

- **HTTP 401**, body `MISSING ACCESS_TOKEN` — no `Authorization` header reached the gateway.
- **HTTP 400** from `/refresh`, empty body — the key is malformed, wrong, or deleted. An empty
  body and a missing body produce the identical response, so log what you sent.
- **HTTP 200 with an `errors` array** — the call failed at the GraphQL layer. A 200 is not a
  success signal on this API; inspect the body. Insufficient key privilege surfaces here, not as
  a 403.
- **5xx** — the gateway itself is down. Check https://status.docontrol.io.

## What is not available

There is no idempotency key, no request-id header, and no rate-limit header. Retries are
unprotected against double-firing — see `conventions/docontrol-conventions.yml` before retrying
any mutation.
