---
name: red-hat-inventory-management
description: Create a new inventory and then list all inventories.
api: openapi/red-hat-inventories-api-openapi.yml
operations:
- createInventory
- listInventories
generated: '2026-10-03'
method: generated
generator: extract-docs-artifacts.py skills (local)
source: openapi/red-hat-inventories-api-openapi.yml ; every operationId checked against the contract
---

# red-hat-inventory-management

Create a new inventory and then list all inventories.

## Steps

1. 1. `createInventory` – requires an authentication header (basicAuth or bearerAuth) and a request body with the inventory fields.
2. 2. `listInventories` – requires an authentication header and may include pagination query parameters.

## Rules

- Authentication: include either a Basic Auth header or a Bearer token in each request.
- Pagination: `listInventories` supports standard pagination parameters as defined by the API.
