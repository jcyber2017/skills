---
name: api-client-query-files
description: 'Create TypeScript React Query files for GET requests with apiClient, typed response models, stable queryKey composition, enabled gating, staleTime, and safe fallback values. Use when generating list/getById query files in frontend/src/queries.'
argument-hint: 'Describe endpoint, return type, params, and fallback policy (e.g. list [] or entity null).'
license: MIT
---

# API Client Query Files

Create standardized, strongly typed query factory files for read-only requests using `apiClient`.

## When to Use
- Creating new GET queries under `frontend/src/queries/**`
- Standardizing query shape across modules
- Migrating ad-hoc query code to the project convention
- Implementing `list*`, `get*ById`, or filtered read queries with consistent typing

## Output Contract
Each generated query file should:
- Import `apiClient` from `@/http/apiClient`
- Import `STALE_TIME` from `@/constants/times`
- Import response types from the module `@/http/.../types`
- Export a default query factory function
- Return an object compatible with React Query options
- Include `retry: false`, `enabled`, and `staleTime: STALE_TIME`

## Procedure
1. Identify query intent and return shape.
2. Define function signature with explicit argument types.
3. Build deterministic `queryKey` with resource id scope and parameters.
4. Implement `queryFn` with explicit Promise return type.
5. Add safe fallback in `.catch(...)` matching return type.
6. Set `retry`, `enabled`, and `staleTime`.
7. Export as default.

## Decision Rules
- If endpoint returns a collection, use `Promise<Entity[]>` and fallback `[]`.
- If endpoint returns one entity, use `Promise<Entity | null>` and fallback `null`.
- Include all function input params in `queryKey` in stable order.
- Keep query factories pure: no side effects and no mutation.

## Templates

### List Query Template
```typescript
import { apiClient } from "@/http/apiClient";
import { STALE_TIME } from "@/constants/times";
import type { Entity } from "@/http/<module>/types";

const listEntitiesQuery = (enabled: boolean) => ({
  queryKey: ["list-entities"],
  queryFn: async (): Promise<Entity[]> => {
    return apiClient.<module>.<resource>.list().catch(() => []);
  },
  retry: false,
  enabled,
  staleTime: STALE_TIME
});

export default listEntitiesQuery;
```

### Get By Id Query Template
```typescript
import { apiClient } from "@/http/apiClient";
import { STALE_TIME } from "@/constants/times";
import type { Entity } from "@/http/<module>/types";

const getEntityByIdQuery = (id: string, enabled: boolean) => ({
  queryKey: ["get-entity-by-id", id],
  queryFn: async (): Promise<Entity | null> => {
    return apiClient.<module>.<resource>.getById(id).catch(() => null);
  },
  retry: false,
  enabled,
  staleTime: STALE_TIME
});

export default getEntityByIdQuery;
```

## Quality Checks
- Types are imported with `import type` and match endpoint response.
- `queryFn` return type and fallback type are identical.
- `queryKey` includes all input args.
- Function and filename naming are consistent (`listXQuery`, `getXByIdQuery`).
- No `any` usage.

## Example Prompts
- `/api-client-query-files Create a list query for treatment purchases returning TreatmentPurchase[] from apiClient.settings.treatmentPurchases.list()`
- `/api-client-query-files Create getById query for food by id with Food type and null fallback`
- `/api-client-query-files Standardize this query file to project conventions and strict typing`
