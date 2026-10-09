---
name: red-hat-cancel-job
description: Cancel a specific job after locating it in the job list.
api: openapi/red-hat-jobs-api-openapi.yml
operations:
- listJobs
- getJob
- cancelJob
generated: '2026-10-03'
method: generated
generator: extract-docs-artifacts.py skills (local)
source: openapi/red-hat-jobs-api-openapi.yml ; every operationId checked against the contract
---

# red-hat-cancel-job

Cancel a specific job after locating it in the job list.

## Steps

1. 1. Use `listJobs` to retrieve the collection of jobs (no required query parameters).
2. 2. Use `getJob` with the `id` field of the target job to view its details.
3. 3. Use `cancelJob` with the same `id` path parameter to cancel the job.

## Rules

- Authentication: include either a `Authorization: Basic <credentials>` header (basicAuth) or a `Authorization: Bearer <token>` header (bearerAuth).
- Idempotency: `cancelJob` is a POST operation that may be retried safely; the API returns the job status after cancellation.
