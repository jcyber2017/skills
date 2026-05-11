---
name: crud-maker
description: 'Orchestrate complete CRUD data layer for React entities. Use when initializing API integration for multiple entities: generates apiClient (if needed), typed entity HTTP endpoints, list/getById queries, and CRUD hooks with full typing, permissions, and state management.'
argument-hint: 'Comma-separated entity list (singular PascalCase: Animal,Provider,Feed) or paste entity requirements'
license: MIT
---

# CRUD Maker

Orchestrate a complete, production-ready CRUD data layer for a React project. Generate an `apiClient`, typed HTTP endpoint modules, React Query files, and hooks in one workflow, eliminating manual step coordination.

## When to Use

Use this skill when the user asks to:
- set up CRUD operations for multiple entities
- initialize the complete API layer (client + endpoints + queries + hooks)
- generate `useEntity`, `listEntityQuery`, and `getEntityByIdQuery` files together
- automate entity scaffolding following project conventions
- migrate ad-hoc API calls to a typed, standardized CRUD layer

## Inputs to Collect

Before executing the plan, collect:
- **Entity list**: singular PascalCase names (e.g. `Animal`, `Provider`, `Treatment`)
- **Module name**: subdirectory path for hooks/queries/http (e.g. `settings`, `animals`)
- **API Base URL**: if apiClient does not exist yet
- **Auth strategy**: token injection method (AWS Cognito pattern, OAuth, custom)
- **Permission keys**: for each entity (e.g. `PERMISSIONS.ANIMAL`, `PERMISSIONS.PROVIDER`)
- **CRUD labels**: Display names for toasts (e.g. `Animal`, `Proveedor`)

If any input is missing, infer from existing project files (check existing hooks, queries, or types).

## Procedure

### Phase 1: Prepare & Inspect
1. Ask the user for the entity list and module.
2. Inspect the project to check if `apiClient.ts` exists and is initialized.
3. Verify React Query, Axios, and types are installed.
4. Check existing `frontend/src/http/<module>/types` for request/response schemas.
5. Check `frontend/src/constants/permissions` for entity permission keys.
6. Confirm `frontend/src/hooks/<module>`, `frontend/src/queries/<module>` directory structure exists or create as needed.

### Phase 2: Initialize API Client (Conditional)
1. If `apiClient.ts` does not exist or lacks typed module getters, trigger the **api-client-generator** skill:
   - Input: target file path, auth provider method, error handling pattern
   - Output: `frontend/src/http/apiClient.ts` with:
     - `axios` setup with request/response interceptors
     - auth token injection for authenticated requests
     - rate limiting and error normalization
     - singleton exports + `useApiClient()` hook
     - typed module getter methods

2. If `apiClient.ts` exists and has the required module getters, skip to Phase 3.

### Phase 3: Generate HTTP Endpoints for Each Entity
For each entity in the list:
1. Trigger the **entity-http-requests-generator** skill:
  - Input: entity name (singular/plural), endpoint path, target module aggregator
  - Output:
    - `frontend/src/http/<module>/<entities>.ts` with typed CRUD methods
    - `frontend/src/http/<module>/index.ts` wired with the new entity module
    - `frontend/src/http/apiClient.ts` module getter wired (create/update)

2. Validate endpoint methods needed by downstream queries/hooks exist:
  - `list()`
  - `getById(id: string)`
  - `create(data)`
  - `update(data)`
  - `remove(id: string)`

### Phase 4: Generate Query Files for Each Entity
For each entity in the list:
1. Trigger the **api-client-query-files** skill for `list<Entity>sQuery.ts`:
   - Input: entity name, module path, endpoint return type (e.g. `Promise<Entity[]>`)
   - Output: `frontend/src/queries/<module>/<entities>/list<Entity>sQuery.ts`

2. Trigger the **api-client-query-files** skill for `get<Entity>ByIdQuery.ts`:
   - Input: entity name, module path, id parameter, fallback type (e.g. `Promise<Entity | null>`)
   - Output: `frontend/src/queries/<module>/<entities>/get<Entity>ByIdQuery.ts`

### Phase 5: Generate CRUD Hooks for Each Entity
For each entity in the list:
1. Collect required types from the project:
   - Response type: `<Entity>` (from `@/http/<module>/types`)
   - Create request type: `<Entity>CreateRequest` (from `@/http/<module>/types`)
   - Update request type: `<Entity>UpdateRequest` (from `@/http/<module>/types`)
   - Permission key: `PERMISSIONS.<ENTITY>` (from `@/constants/permissions`)
   - CRUD resource key: `CRUD_RESOURCES.<ENTITY>` (from `@/constants/permissionMessages`)
   - Spanish label: user-provided or inferred

2. Trigger the **crud-hooks** skill:
   - Input: entity name, module, permission keys, apiClient path, Spanish label
   - Output: `frontend/src/hooks/<module>/use<Entity>s.ts` with:
     - list and getById queries (useQuery)
     - create, update, remove mutations (direct apiClient calls)
     - permission guards + loading state aggregation
     - post-mutation refetch logic

### Phase 6: Validate & Summarize
1. Check all generated files for:
   - TypeScript compilation errors
   - Import resolution (no missing types or modules)
   - Consistent naming across hooks, queries, and types
   - No `any` usage without justification

2. Generate a summary report with:
  - Files created/modified (apiClient + 1 HTTP entity module × entity count + query/hook files)
   - Key features implemented per phase
   - Any assumptions or inferred conventions
   - Next steps (e.g. adding new endpoints, creating forms, tests)

## Decision Points

- **API Client existence**:
  - If it exists with the required module getters, reuse it.
  - If it exists but lacks typed getters, ask whether to extend or regenerate.
  - If missing, generate it.

- **Module structure**:
  - If `frontend/src/queries/<module>/<entities>/` does not exist, create the directory tree.
  - If `frontend/src/hooks/<module>/` does not exist, create it.

- **Type inference**:
  - If response types are missing from `@/http/<module>/types`, ask the user to define them first or skip that entity.
  - If permission keys are missing, use a fallback pattern and flag for manual addition.

- **Permission handling**:
  - If permissions are not defined, generate hooks without permission guards and note this for manual setup.
  - All hooks include `canCreate`, `canEdit`, `canView`, `canDelete` checks.

## Quality Checklist

Before finishing, ensure:
- All entities have corresponding `use<Entity>s.ts` hooks
- All entities have corresponding HTTP endpoint modules under `frontend/src/http/<module>/`
- All hooks have `list<Entity>sQuery.ts` and `get<Entity>ByIdQuery.ts` files
- All files are TypeScript-valid with no compilation errors
- No `any` types used unless explicitly justified
- Query keys follow the naming pattern (`list-<entities>`, `get-<entity>-by-id`)
- Mutations call `refetch` hooks in the correct order (getById then list)
- Toast messages are in Spanish and match entity labels
- Permission imports resolve correctly
- All imports use project aliases (`@/`) and match lint rules

## Output Requirements

Return a concise report with:
- **Files created**: list each file path and its purpose
- **API Client status**: created, extended, or reused
- **Entities processed**: count and list names
- **Key features**: 
  - apiClient configured with auth and rate limiting
  - typed HTTP entity endpoints wired in module aggregator and apiClient
  - 2 query files per entity (list + getById)
  - 1 hook per entity with full CRUD + permissions
  - post-mutation refetch strategy
  - error handling and toast notifications
- **Assumptions made**: inferred module names, permission keys, or type patterns
- **Next steps**: suggested follow-up tasks (e.g. define missing types, create forms, add tests)
- **Example usage**: show how to import and use one hook in a component

## Example Prompts

- `/crud-maker Animal, Provider, Feed in the animals module`
- `/crud-maker Set up CRUD for Treatment, Food, Supplier with permissions and full typing`
- `/crud-maker Initialize complete API layer for Entity1, Entity2, Entity3`
- `/crud-maker Given entities: AnimalGroup, AnimalType, HealthRecord — generate queries and hooks`
