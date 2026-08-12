---
name: Dispatch a WellCube device action and poll the resulting job
description: >-
  Submit an asynchronous command to a live WellCube installation, then poll for job state and paginate
  the results — the highest-consequence flow in the Delos API surface.
api: openapi/delos-wellcube-cloud-be-openapi.yml
base_url: https://cloud.wellcube.io/api/v1
operations:
  - globalExecuteNVA
  - globalJobsShow
  - globalJobsGetResults
  - actionSubmit
  - actionShow
generated: '2026-08-12'
method: generated
source: openapi/delos-wellcube-cloud-be-openapi.yml
---

# Dispatch a device action and poll the job

> **Read this first.** `globalExecuteNVA` sends a command to physical equipment on a live building
> installation — air purifiers and sensor devices in an occupied space. There is **no idempotency key
> anywhere in this API**, so a retried dispatch is a second dispatch, and the API returns **HTTP 200
> even when it fails**, so a naive client cannot tell a failed dispatch from a successful one without
> parsing the body. Do not run this flow unattended. Confirm with a human before dispatching, and
> never blind-retry.

Authenticate first (see `delos-authenticate-and-list-installations.md`) and send the token as the
`Authorization` header. Switch on `body.status` (`1` ok, `0` error), never on the HTTP status line.

## Step 1 — dispatch

```
POST /global/execute-nva
Authorization: <token>
```

`globalExecuteNVA` accepts the command and hands back a job identifier for polling. Failure codes:

| Code | Response key | Meaning |
|---|---|---|
| `NOT_EXISTS` | `x-not-exists` | the target installation or entity is unknown |
| `NOT_FOUND` | `x-not-found` | distinct from `NOT_EXISTS` — the spec declares both on this operation and does not explain the difference |
| `PERMISSION_DENIED` | `x-permission-denied` | you are not entitled to command this installation |

Both `NOT_EXISTS` and `NOT_FOUND` are terminal for the dispatch. Stop; do not retry either.

If the request times out or the connection drops, **do not resend.** You cannot tell whether the
command reached the equipment. Inspect job state instead, and escalate to a human if that is
inconclusive.

## Step 2 — poll for state

`globalJobsShow` — `GET /global/jobs/{jobId}` (`jobId` is a UUID).

The job entity is **not declared as a component schema** in the OpenAPI, so its state values are not
enumerated in the contract. Poll until the response stops indicating in-progress, and impose your own
ceiling on attempts and elapsed time. Because no rate-limit headers are returned, use a conservative
backoff — start around a second and grow it; you will get no throttling signal before you are cut off.

Failures: `NOT_EXISTS` (unknown job) and `PERMISSION_DENIED` (not yours).

## Step 3 — read the results

`globalJobsGetResults` — `GET /global/jobs/{jobId}/results`.

This is the only non-list operation that is paginated. It takes the shared `limit` and `offset`
components, plus a `noun` query filter that narrows results to one entity kind.

No total count and no next link is declared, so page until a request returns fewer rows than `limit`.

Failures here add one beyond the usual pair: `ACTION_NOT_ALLOWED` via `x-action-not-allowed`, which
means the job exists but its results are not readable in its current state — poll `globalJobsShow`
again rather than re-requesting results.

## The per-action variant

`actions` is the same submit-then-read shape at single-action granularity:

- `actionSubmit` — `POST /actions/{actionId}`
- `actionShow` — `GET /actions/{actionId}`

The `Action` entity carries `id`, `type`, `data`, `isExpired`, `expiresAt` and `createdAt`. Check
`isExpired` before submitting: an expired action returns `EXPIRED` via the shared `ExpiredError`
component, and there is no renew operation for actions the way there is for invitations.

One contract gap worth knowing: `data` is listed in the schema's `required` array but has **no
property definition**, so the spec does not tell you what shape an action payload takes. You will need
a worked example from Delos to populate it correctly. Do not guess at it against a live installation.

## Safety checklist before any dispatch

- [ ] A human has approved this specific command against this specific installation.
- [ ] The installation's `connected` flag is `true` (from `installationsList`).
- [ ] You have recorded the `jobId` so a timeout can be investigated rather than retried.
- [ ] Your retry policy on this flow is: **do not retry**. Re-read state instead.
