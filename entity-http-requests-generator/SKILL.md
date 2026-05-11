---
name: entity-http-requests-generator
description: 'Generate a typed entity HTTP module for frontend settings APIs (for example foods, operators, customers), then wire it into frontend/src/http/module/index.ts and frontend/src/http/apiClient.ts. Use when adding a new module entity endpoint or standardizing entity request modules in TypeScript.'
argument-hint: 'Entity name (singular/plural), endpoint path, and whether to create/update module aggregator'
user-invocable: true
license: MIT
---

# Entity HTTP Requests Generator

Create or update a strongly typed entity request module in the frontend HTTP layer, then connect it to the module aggregator and API client.

## When To Use

Use this skill when the user asks to:
- add a new module entity like `frontend/src/http/module/foods.ts`
- create a typed CRUD request file for an entity in a module
- wire a new entity module into `frontend/src/http/module/index.ts`
- expose the module through `frontend/src/http/apiClient.ts`
- bootstrap a module called general file when no module aggregator exists

## Inputs

Collect or infer these inputs:
- entity file name in plural snake/camel form used by endpoints (example: `foods`, `operators`, `customers`)
- type names in `frontend/src/http/<module>/types.ts`:
  - `<Entity>`
  - `<Entity>CreateRequest`
  - `<Entity>UpdateRequest`
- endpoint path segment (default: same as entity file name)
- target module aggregator path (default: `frontend/src/http/<module>/index.ts`)
- target API client path (default: `frontend/src/http/apiClient.ts`)

If one of these is missing, infer from existing sibling files in `frontend/src/http/<module>/`.

## Procedure

1. Inspect conventions before editing.
- Review existing entity files like `foods.ts`, `operators.ts`, `customers.ts`.
- Match import alias style (`@/...`), return style (`(await axios.<method>()).data`), and default export pattern.

2. Create or update the entity module file.
- File path: `frontend/src/http/<module>/<entity>.ts`
- Use this structure:
  - `import type { AxiosInstance } from "axios";`
  - import `ObjectResponse`, `CreateRequest`, `UpdateRequest` aliases from `@/http/<module>/types`
  - export a factory: `const <entity> = (axios: AxiosInstance, prefix = "") => ({ ... })`
  - include methods:
    - `getById(id: string): Promise<ObjectResponse>`
    - `list(): Promise<ObjectResponse[]>`
    - `create(data: CreateRequest)`
    - `update(data: UpdateRequest)`
    - `remove(id: string)`
  - each method must call `${prefix}/<endpoint>` (or `${prefix}/<endpoint>/${id}`) and return `.data`
  - `export default <entity>;`

3. Wire into the module aggregator.
- Preferred file: `frontend/src/http/<module>/index.ts`
- Add import: `import <entity> from "@/http/<module>/<entity>";`
- Add property in returned object: `<entity>: <entity>(axios, prefix),`

4. Handle missing aggregator module.
- If `frontend/src/http/<module>/index.ts` does not exist:
  - create a general module file at that path
  - include `AxiosInstance` and return object with at least the new entity module
  - preserve extensible shape so more entities can be added later

5. Attach module to API client.
- Open `frontend/src/http/apiClient.ts`.
- Ensure module import exists: `import api<Module> from "@/http/<module>";`
- Ensure getter exists and returns module wired with axios + prefix:
  - `get <module>() { return api<Module>(this.axios, "/<module>"); }`
- If the project uses another module name (not `<module>`), attach the generated module under the established naming convention.

6. Validate after edits.
- Confirm TypeScript types resolve for `<Entity>`, `<Entity>CreateRequest`, `<Entity>UpdateRequest`.
- Ensure module default export and aggregator property names are consistent.
- Ensure endpoint strings match entity naming and existing REST pattern.
- Run lint/type checks when available and fix introduced errors.

## Decision Points

- Endpoint naming mismatch:
  - If type names are singular but endpoint is plural, keep existing API endpoint convention and only adapt aliases.
- Prefix scope choice:
  - For global module resources, use `prefix`.
- Missing type definitions:
  - If entity types are absent in `types.ts`, create them or ask user for schema before finalizing.
- No module architecture present:
  - Create a minimal general aggregator first, then attach entity module and API client getter.

## Quality Checks

Before finishing, verify:
- entity file follows sibling module style exactly
- no `any` introduced in request or response types
- CRUD methods and paths are complete and correctly typed
- module aggregator includes the entity and compiles
- `apiClient` exposes the module getter consistently
- new code is minimal, organized, and production-ready

## Output Format

When done, report:
- files created/updated
- generated entity methods and endpoint paths
- aggregator and apiClient wiring completed
- assumptions or unresolved schema/type gaps
