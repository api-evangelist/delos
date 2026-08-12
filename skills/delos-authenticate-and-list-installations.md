---
name: Authenticate against the WellCube Cloud BE API and list installations
description: >-
  Obtain a WellCube session token, verify it, and page through the installations (deployed WellCube
  systems) visible to the caller — including their live connection state, network identity and
  associated products.
api: openapi/delos-wellcube-cloud-be-openapi.yml
base_url: https://cloud.wellcube.io/api/v1
operations:
  - limitedSessionCreate
  - limitedSessionRefresh
  - sessionValidate
  - installationsList
  - adminInstallationsList
  - adminInstallationsListAssociatedProducts
generated: '2026-08-12'
method: generated
source: openapi/delos-wellcube-cloud-be-openapi.yml
---

# Authenticate and list WellCube installations

## Before you start — three things this API does differently

1. **Every response is HTTP 200, including failures.** Verified live on 2026-08-12. Do not branch on
   the HTTP status line. Branch on `body.status`: `1` means success and the payload is in `body.data`;
   `0` means failure and the payload is in `body.error`.
2. **There is no idempotency key.** Nothing in this skill writes, so retries are safe here — but do not
   carry that assumption into the write skills.
3. **There are no rate-limit headers.** Pace yourself; you will get no signal before you are throttled.

## Step 1 — get a token

Prefer `limitedSessionCreate` over `sessionCreate`. Both accept email + password, but `sessionCreate`
returns a bare `jwt` and the provider's own spec labels that response *"Deprecated create session
response"*. `limitedSessionCreate` returns the typed `AccessData` pair and is the only flow with a
refresh operation.

```
POST /limited-sessions
Content-Type: application/json

{ "data": { "email": "<email>", "password": "<password>" } }
```

Success gives `{ status: 1, data: { accessToken, refreshToken } }`.

Failure gives `{ status: 0, error: { code: "AUTHENTICATION_FAILED", fields: { email: "INVALID",
password: "INVALID" } } }`. The `fields` object tells you which credential was rejected, and
`fields.status: "NOT_ACTIVE_USER"` means the account exists but is disabled — do not retry that one
with different credentials.

Send the `accessToken` on every subsequent call as the `Authorization` request header. If you omit it
you get `{ status: 0, error: { code: "FORMAT_ERROR", fields: { token: "REQUIRED" } } }` — note that
`FORMAT_ERROR` is undocumented, so match on it defensively.

When the token expires, call `limitedSessionRefresh` (`POST /limited-sessions/refresh`) rather than
re-authenticating with the password.

## Step 2 — confirm the token is live

`sessionValidate` (`GET /sessions/validate`) exists specifically for this. It distinguishes three
failure modes that matter:

- `x-wrong-token` → `WRONG_TOKEN`: the token is malformed or not ours.
- `x-user-not-valid-token` → `USER_NOT_VALID`: the user encoded in the token no longer resolves.
- `x-user-not-active-token` → `USER_NOT_ACTIVE`: the user resolves but is deactivated.

Only the first is worth retrying after a refresh.

## Step 3 — list installations

For a normal caller use `installationsList` (`GET /installations`). For an administrator use
`adminInstallationsList` (`GET /admin/installations`), which exposes more filters.

Shared paging and ordering parameters, declared once as reusable components and referenced by both:

| Parameter | Type | Notes |
|---|---|---|
| `limit` | integer | example `20`. No default, minimum or maximum is declared. |
| `offset` | integer | example `0`. Offset paging. |
| `order` | enum | `ASC` or `DESC`. |
| `sortedBy` | enum | `installationsList`: `id`, `name`, `createdAt`, `updatedAt`. `adminInstallationsList`: `id`, `localUserName`, `createdAt`, `updatedAt`. |

Filters on `installationsList`: `productId`, `productIsPrimary`.
Filters on `adminInstallationsList`: `search`, `connected`, `productName`, `tags` (array), `userId`.

**Detecting the end of the collection.** No response declares a total count, a next link, or a
has-more flag. Page until a request returns fewer items than `limit`; that is the only termination
signal this contract gives you.

## Step 4 — read what you got back

Each `Installation` carries `id`, `name`, `macAddress`, `ipAddress`, `ssid`, `connected`, `tags[]`,
`createdAt`, `updatedAt`, plus two lists that are **always** embedded (there is no `expand` parameter
and no way to suppress them):

- `localAccounts[]` — the on-device user identities bound to this installation.
- `translators[]` — `{ id, name }` — the protocol adapters bridging devices into the cloud.

`connected` is the live health signal: it is what tells you whether a site's WellCube system is
currently reachable.

## Step 5 — which products a site runs

`adminInstallationsListAssociatedProducts` (`GET /admin/installations/{installationId}/products`)
returns the products bound to one installation. `installationId` is a UUID.

For utilisation figures use `adminInstallationStats`
(`GET /admin/installations/{installationId}/stats`). It returns `x-wrong-id` (`WRONG_ID`) for a
malformed id — distinct from `x-permission-denied` (`PERMISSION_DENIED`), which means the id was fine
but you are not entitled to it. Twenty-eight of the thirty-nine operations can return
`PERMISSION_DENIED`, so handle it once, centrally.

## Error handling summary

Match on `error.code`, not on the response key and not on the HTTP status:

| Code | Meaning | Action |
|---|---|---|
| `FORMAT_ERROR` | missing/invalid `Authorization` header (undocumented) | attach or refresh the token |
| `AUTHENTICATION_FAILED` | bad credentials at login | do not retry with the same credentials |
| `WRONG_TOKEN` | malformed token | refresh |
| `USER_NOT_VALID` / `USER_NOT_ACTIVE` | user gone or disabled | stop; needs a human |
| `PERMISSION_DENIED` | not entitled to this resource | stop; do not enumerate around it |
| `WRONG_ID` | malformed identifier | fix the id |
| `NOT_EXISTS` / `NOT_FOUND` | resource absent | stop |
