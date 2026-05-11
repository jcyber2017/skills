---
name: crud-hooks
description: 'Create TypeScript CRUD React hooks with React Query caching for GET requests and direct apiClient mutations (create, update, delete). Use when generating use<Entity>s.ts hooks and their paired list/getById query files. Handles permissions, loading state, toast notifications, and post-mutation refetch.'
argument-hint: 'Entity name (singular), module path, permission key, CRUD_RESOURCES key, apiClient path, and Spanish display label.'
license: MIT
---

# CRUD Hooks Generator

Generates a complete, typed CRUD hook plus its two React Query files following project conventions.

## When to Use
- Creating a new `use<Entity>s.ts` hook under `frontend/src/hooks/<module>/`
- Adding list + getById queries under `frontend/src/queries/<module>/<entity>/`
- Standardizing an existing mutation-only hook to follow project conventions
- Any CRUD operation involving settings or feature entities with permissions

## Output: 3 files

| File | Purpose |
|------|---------|
| `frontend/src/hooks/<module>/use<Entity>s.ts` | Hook: state, queries, mutations, permissions |
| `frontend/src/queries/<module>/<entity>/list<Entity>sQuery.ts` | React Query list factory |
| `frontend/src/queries/<module>/<entity>/get<Entity>ByIdQuery.ts` | React Query getById factory |

## Procedure

### Step 1 – Gather inputs
Identify from the request or existing code:
- `Entity` – PascalCase singular (e.g. `Provider`)
- `entity` – camelCase singular (e.g. `provider`)
- `entities` – camelCase plural (e.g. `providers`)
- `module` – subdirectory under `hooks/` and `queries/` (e.g. `settings`)
- `apiClient.<module>.<entities>` – apiClient path (e.g. `apiClient.settings.providers`)
- `PERMISSIONS.<ENTITY>` – key from `@/constants/permissions`
- `CRUD_RESOURCES.<ENTITY>` – key from `@/constants/permissionMessages`
- Spanish display label for toasts (e.g. `Proveedor`)
- `<Entity>CreateRequest`, `<Entity>UpdateRequest` – types from `@/http/<module>/types`
- `<Entity>` response type – from `@/http/<module>/types`

If any input is missing, infer from existing files in the same module. Check sibling hooks and queries for conventions.

### Step 2 – Create list query file
Path: `frontend/src/queries/<module>/<entities>/list<Entity>sQuery.ts`

Follow the [list query template](#list-query-template).

Key rules:
- `queryKey` must looks like this: `['list-<entities>']`
- `queryFn` returns `Promise<Entity[]>` with `.catch(() => [])`
- Include `retry: false`, `enabled`, `staleTime: STALE_TIME`

### Step 3 – Create getById query file
Path: `frontend/src/queries/<module>/<entities>/get<Entity>ByIdQuery.ts`

Follow the [getById query template](#getbyid-query-template).

Key rules:
- `queryKey`: `['get-<entity>-by-id', id]`
- `queryFn` returns `Promise<Entity | null>` with `.catch(() => null)`
- Accepts `(id: string, enabled: boolean)`

### Step 4 – Create hook file
Path: `frontend/src/hooks/<module>/use<Entity>s.ts`

Follow the [hook template](#hook-template).

Key rules:
- `useQuery` for list and getById only — no `useMutation`
- `create`, `update`, `remove` call `apiClient` directly, then `refetch`
- After `create`: `await refetchItems()` only
- After `update`: `await refetchItem()` then `await refetchItems()`
- After `remove`: `await refetchItem()` then `await refetchItems()`
- Always `canDoCrud(enabled, canXxx, CRUD_ACTIONS.XXX)` guards mutations
- `loading` aggregates mutation + both query loading/fetching states
- Return `items ?? []` (never undefined) and `item` (may be undefined)

### Step 5 – Verify
- All imports resolve; no `any`
- Toast messages are in Spanish and match entity
- `remove` parameter name uses `<entity>Id`
- `setLoading(false)` is always in `finally`

---

## Templates

### List Query Template
```typescript
import { apiClient } from "@/http/apiClient";
import { STALE_TIME } from "@/constants/times";
import type { Entity } from "@/http/<module>/types";

const listEntitiesQuery = (enabled: boolean) => ({
  queryKey: ['list-entities'],
  queryFn: async (): Promise<Entity[]> => {
    return apiClient.<module>.entities.list().catch(() => []);
  },
  retry: false,
  enabled,
  staleTime: STALE_TIME
});

export default listEntitiesQuery;
```

### GetById Query Template
```typescript
import { apiClient } from "@/http/apiClient";
import { STALE_TIME } from "@/constants/times";
import type { Entity } from "@/http/<module>/types";

const getEntityByIdQuery = (id: string, enabled: boolean) => ({
  queryKey: ['get-entity-by-id', id],
  queryFn: async (): Promise<Entity | null> => {
    return apiClient.<module>.entities.getById(id).catch(() => null);
  },
  retry: false,
  enabled,
  staleTime: STALE_TIME
});

export default getEntityByIdQuery;
```

### Hook Template
```typescript
import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';

import { useToast } from '@/hooks/use-toast';
import getEntityByIdQuery from '@/queries/<module>/entities/getEntityByIdQuery';
import listEntitiesQuery from '@/queries/<module>/entities/listEntitiesQuery';
import { apiClient } from '@/http/apiClient';
import { PERMISSIONS } from '@/constants/permissions';
import { useFeaturePermissions } from '@/hooks/permissions/useFeaturePermissions';
import canDoCrudInThisResource from '@/functions/canDoCrudInThisResource';
import { CRUD_ACTIONS } from '@/constants/crudMessages';
import { CRUD_RESOURCES } from '@/constants/permissionMessages';
import type { EntityCreateRequest, EntityUpdateRequest } from '@/http/<module>/types';

export function useEntities(id?: string) {
  const [loading, setLoading] = useState<boolean>(false);
  const { toast } = useToast();
  const { canView, canCreate, canEdit, canDelete } = useFeaturePermissions(PERMISSIONS.ENTITIES);

  const {
    data: items,
    isLoading: isLoadingItems,
    isFetching: isFetchingItems,
    refetch: refetchItems
  } = useQuery(listEntitiesQuery(canView));
  const {
    data: item,
    isLoading: isLoadingItem,
    isFetching: isFetchingItem,
    refetch: refetchItem
  } = useQuery(getEntityByIdQuery(id ?? '', !!id && canView));

  const canDoCrud = canDoCrudInThisResource(toast, CRUD_RESOURCES.ENTITIES);

  const create = async (creationData: EntityCreateRequest) => {
    if (loading || !canDoCrud(true, canCreate, CRUD_ACTIONS.CREATE)) return;
    try {
      setLoading(true);
      await apiClient.<module>.entities.create(creationData);
      toast({
        title: 'Éxito',
        description: '<Label> registrado/a correctamente',
      });
      // Force refetch after creation
      await refetchItems();
    } catch (error) {
      console.error('Error creating <entity>:', error);
      throw error;
    } finally {
      setLoading(false);
    }
  };

  const update = async (updateData: EntityUpdateRequest) => {
    if (loading || !canDoCrud(true, canEdit, CRUD_ACTIONS.UPDATE)) return;
    try {
      setLoading(true);
      await apiClient.<module>.entities.update(updateData);
      toast({
        title: 'Éxito',
        description: '<Label> actualizado/a correctamente',
      });
      // Force refetch after update
      await refetchItem();
      await refetchItems();
    } catch (error) {
      console.error('Error updating <entity>:', error);
      throw error;
    } finally {
      setLoading(false);
    }
  };

  const remove = async (entityId: string): Promise<void> => {
    if (loading || !canDoCrud(true, canDelete, CRUD_ACTIONS.REMOVE)) return;
    try {
      setLoading(true);
      await apiClient.<module>.entities.remove(entityId);
      toast({
        title: 'Éxito',
        description: '<Label> eliminado/a correctamente',
      });
      // Force nullify item and refetch after deletion
      await refetchItem();
      await refetchItems();
    } catch (error) {
      console.error('Error deleting <entity>:', error);
      throw error;
    } finally {
      setLoading(false);
    }
  };

  return {
    item,
    items: items ?? [],
    loading: loading || isLoadingItems || isFetchingItems || isLoadingItem || isFetchingItem,
    create,
    update,
    remove,
  };
}
```

## Quality Checks
- [ ] `queryFn` fallbacks match return types (`[]` for list, `null` for getById)
- [ ] No `useMutation` — mutations use `apiClient` directly
- [ ] All mutations are guarded with `canDoCrud` before `setLoading(true)`
- [ ] `setLoading(false)` always in `finally` block
- [ ] `refetchItem` called before `refetchItems` in `update` and `remove`
- [ ] Toast messages are in Spanish
- [ ] `remove` param is named `<entity>Id`
- [ ] Return object spreads `items ?? []` not raw `items`
- [ ] All types imported with `import type`

## Example Prompts
- `/crud-hooks Create CRUD hook for Corral entity in settings module`
- `/crud-hooks Add useBreeds hook following the same pattern as useProviders`
- `/crud-hooks Generate hook and queries for TreatmentType in the settings module`
