# nxt API

The nxt API lets a user create and manage tasks, retrieve their next recommended task, and manage the personal context nxt uses when organising their work.

The API is currently versioned as `v1` and is served from:

```text
https://app.nxt.do/api/public/v1
```

The complete machine-readable contract is in [`openapi.yaml`](openapi.yaml).

## Get started

1. Download nxt from the [Apple App Store](https://apps.apple.com/app/nxt-ai-to-do-lists-tasks/id6717590335) or [Google Play](https://play.google.com/store/apps/details?id=com.nxt.app), then create your account in the app.
2. Contact [nxt support](mailto:support@nxt.do) to request an API token. API token provisioning is currently handled manually.
3. Send the token using the Bearer authentication scheme:

```bash
curl https://app.nxt.do/api/public/v1/tasks \
  --header "Authorization: Bearer $NXT_API_TOKEN"
```

API tokens begin with `nxt_live_`. Treat them like passwords. Contact [nxt support](mailto:hello@nxt.do) if a token needs to be revoked or replaced.

## Create a task

Task creation accepts natural-language text and is processed asynchronously:

```bash
curl https://app.nxt.do/api/public/v1/tasks \
  --request POST \
  --header "Authorization: Bearer $NXT_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"task":{"text":"Remind me to buy groceries tomorrow at 5pm"},"idempotency_key":"groceries-2026-07-16"}'
```

The API responds with `202 Accepted`, a `Location` header, and a task request:

```json
{
  "task_request": {
    "id": 42,
    "status": "draft",
    "poll_url": "https://app.nxt.do/api/public/v1/task_requests/42",
    "created_at": "2026-07-16T12:00:00Z",
    "updated_at": "2026-07-16T12:00:00Z"
  }
}
```

Poll `task_request.poll_url` until the status is `active` or `failed`. An active request contains the created task or tasks. Reusing the same `idempotency_key` with the same input returns the original request instead of creating duplicates; using it with different input returns `409 Conflict`.

## Tasks

List active tasks:

```bash
curl "https://app.nxt.do/api/public/v1/tasks?page=1&page_size=50" \
  --header "Authorization: Bearer $NXT_API_TOKEN"
```

Use `status=active`, `completed`, `deleted`, or `all` to select another task state.

Update a task using explicit local dates and 24-hour times:

```bash
curl https://app.nxt.do/api/public/v1/tasks/123 \
  --request PATCH \
  --header "Authorization: Bearer $NXT_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"task":{"text":"Buy groceries","due_date":"2026-07-17","due_time":"17:00"}}'
```

Deleting a task is a soft deletion and returns `204 No Content`.

## Next task

```bash
curl -i https://app.nxt.do/api/public/v1/next-task \
  --header "Authorization: Bearer $NXT_API_TOKEN"
```

- `200 OK` returns the current recommendation.
- `202 Accepted` means nxt is calculating one. Wait for the number of seconds in `Retry-After` and retry.
- `204 No Content` means no task is currently available.

Public API requests never trigger mobile push notifications.

## Context

Context is personal information that helps nxt prioritise tasks appropriately:

```bash
curl https://app.nxt.do/api/public/v1/contexts \
  --request POST \
  --header "Authorization: Bearer $NXT_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"context":{"text":"I work from home on Fridays."}}'
```

Context text must be present and cannot exceed 1,000 characters.

## Errors and limits

Errors use a consistent envelope:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "The request could not be processed.",
    "details": {
      "text": ["can't be blank"]
    }
  }
}
```

The API permits 120 requests per token per minute. A client exceeding the limit receives `429 Too Many Requests`.

All resources are scoped to the token owner. Requests for missing resources or resources belonging to another account both return `404 Not Found`.

## Versioning

Breaking changes will be introduced under a new URL version. Additive fields may be added to existing responses, so clients should ignore fields they do not recognise.
