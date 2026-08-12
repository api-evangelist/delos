---
name: Invite a user to a WellCube product and manage the invitation lifecycle
description: >-
  Create, inspect, confirm, renew and revoke the product invitations that grant a user access to a
  WellCube/Darwin product, and read back the resulting entitlements.
api: openapi/delos-wellcube-cloud-be-openapi.yml
base_url: https://cloud.wellcube.io/api/v1
operations:
  - userProductInvitationCreate
  - userProductInvitationShow
  - userProductInvitationConfirm
  - userProductInvitationRenew
  - userProductInvitationRevoke
  - userProductsList
  - adminUserProductsList
  - adminUserProductsCreate
  - adminUserProductsUpdate
  - adminUserProductsDelete
generated: '2026-08-12'
method: generated
source: openapi/delos-wellcube-cloud-be-openapi.yml
---

# Invite a user to a WellCube product

Authenticate first — see `delos-authenticate-and-list-installations.md`. Send the token as the
`Authorization` header on every call. Remember: **HTTP status is always 200**; read `body.status`
(`1` ok, `0` error) and switch on `body.error.code`.

**This skill writes.** The API publishes no idempotency key, so a retried create is a second create.
Before retrying any failed write, read the current state back with `userProductInvitationShow` or
`userProductsList` and decide from that.

## Step 1 — create the invitation

```
POST /users/product-invitations
Authorization: <token>
```

`userProductInvitationCreate` returns a `ProductInvitation`:

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | the `invitationId` for every later call |
| `userId` | uuid | invitee |
| `email` | string | invitee address |
| `productName` | string | the product being granted |
| `invitedBy` | uuid | the inviting user |
| `isExpired` | boolean | |
| `expiresAt` | date-time | |
| `createdAt` | date-time | |

Failures specific to this operation:

- `NOT_EXISTS` via `x-product-not-exists` — the named product does not exist. Confirm it first with
  `productList` (`GET /products`, filterable by `name`).
- `NOT_UNIQUE` via `x-user-not-unique` — the invitee is ambiguous or already invited. **Do not retry.**
  Fetch the existing invitation instead.
- `PERMISSION_DENIED` — you may not invite to this product.

## Step 2 — inspect it

`userProductInvitationShow` (`GET /users/product-invitations/{invitationId}`). Returns `NOT_EXISTS`
via `x-action-not-exists` for an unknown id. Check `isExpired` and `expiresAt` before acting on it.

## Step 3 — the three state transitions

All three are `POST` on a sub-path of the invitation and all take `invitationId` as a UUID path
parameter.

**Confirm** — `userProductInvitationConfirm`, `POST /users/product-invitations/{invitationId}/confirm`.
This is the operation that actually grants the entitlement. It has the richest failure set in the
surface, and each one means something different:

| Code | Response key | Meaning |
|---|---|---|
| `EXPIRED` | `x-action-expired` | the invitation lapsed — renew it, do not re-create |
| `NOT_EXISTS` | `x-action-not-exists` | no such invitation |
| `NOT_EXISTS` | `x-product-not-exists` | the product was removed after the invite was issued |
| `NOT_EXISTS` | `x-user-not-exists` | the invitee no longer exists |
| `PERMISSION_DENIED` | `x-permission-denied` | not yours to confirm |

**Renew** — `userProductInvitationRenew`,
`POST /users/product-invitations/{invitationId}/renew`. The correct response to `EXPIRED`. Extends the
existing invitation rather than minting a new one, which keeps `id` stable and avoids the
`NOT_UNIQUE` collision a re-create would cause.

**Revoke** — `userProductInvitationRevoke`,
`POST /users/product-invitations/{invitationId}/revoke`. Withdraws it. Treat as irreversible: there is
no un-revoke operation.

## Step 4 — verify the entitlement landed

`userProductsList` (`GET /users/{userId}/products`) lists the products a user holds. `userId` is a
UUID. This is the read-back that closes the loop after a confirm, and the safe way to check state
before any retry.

## Administrative path — grant directly, no invitation

An admin caller can bypass the invitation flow entirely:

- `adminUserProductsList` — `GET /admin/users/{userId}/products`
- `adminUserProductsCreate` — `POST /admin/users/{userId}/products/{productId}`
- `adminUserProductsUpdate` — `PUT /admin/users/{userId}/products/{productId}`
- `adminUserProductsDelete` — `DELETE /admin/users/{userId}/products/{productId}`

Two cautions.

First, a contract inconsistency: on these four operations `productId` is declared
`type: string, format: string`, while the same identifier is declared `format: uuid` on
`adminProductShow`, `adminProductUpdate`, `adminProductDelete` and `adminProductStats`. Send the UUID
you got from `adminProductList`; do not infer a second identifier format from the looser declaration.

Second, `adminUserProductsDelete` removes an entitlement with no confirmation step, no soft-delete and
no idempotency key. Read the entitlement back with `adminUserProductsList` first, and never retry a
delete that appeared to fail — re-read instead.

`adminUserProductsUpdate` additionally returns `ACTION_NOT_ALLOWED` via `x-action-not-allowed` when
the transition itself is forbidden, which is distinct from `PERMISSION_DENIED` (you lack rights) and
from `NOT_EXISTS` (nothing there). Do not collapse the three.
