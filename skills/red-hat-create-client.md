---
name: red-hat-create-client
description: Create a new client in a realm after optionally listing existing clients.
api: openapi/red-hat-clients-api-openapi.yml
operations:
- listClients
- createClient
generated: '2026-10-03'
method: generated
generator: extract-docs-artifacts.py skills (local)
source: openapi/red-hat-clients-api-openapi.yml ; every operationId checked against the contract
---

# red-hat-create-client

Create a new client in a realm after optionally listing existing clients.

## Steps

1. 1. Call `listClients` with the required path parameter `realm` and any query parameters the contract defines.
2. 2. Call `createClient` with the required path parameter `realm`, a request body containing the client representation, and include any required headers.

## Rules

- Authentication: Provide either a `basicAuth` or `bearerAuth` HTTP Authorization header as defined by the API.
- Idempotency: The `createClient` operation is not documented as idempotent; callers should avoid duplicate requests.
